# kintera audit, sihe's half: thermo, equilibrium, vapors, utils, units, diffusion, top-level src

Read at `26e359edc1b28f8bb4f75b13855c663f6e14a6d7` (v3/stack/K5 head, on v3/base6 = `431fd144` = chengcli/kintera main, v2.6.11). Findings only: nothing below was built or run except the searches whose commands and outputs are in the appendix. "Bitwise" means the same floating-point operations in the same order, so I expect no result to move; it is a prediction until the row is built and gated. Jiachen's half (kinetics, photochem, math) is not covered; where a copy of one of my sites lives in his directories I say so and do not count it.

Lines audited (`find src/<dir> -type f | xargs cat | wc -l`): thermo 4691, equilibrium 673, vapors 203, utils 1891, units 534, diffusion 107, top-level src files 1444; total 9543.

## Fix first (affects the K5 PR's own CI)

| id | what | sites file:line@26e359ed | which stays | others become | expected result | net lines |
|---|---|---|---|---|---|---|
| F1 | `fallback_version = "2.5.13"` is below the newest tag v2.6.11, so `tests/test_version_fallback.py` (a `git archive HEAD` of the tree reports `2.5.13+g<sha>`) fails on every untagged commit; it passes on v3/base6 only because 431fd144 is the tag itself | pyproject.toml:86; tests/test_version_fallback.py:44 | the test | `fallback_version = "2.6.11"` (tests/test_version_floor.py:68 only requires the floor to be ≤ the newest tag) | test_version_fallback passes at any commit on top of v2.6.11; nothing numeric | 0 |

## Question 1: things defined in more than one place

| id | what | sites file:line@26e359ed | which stays | others become | expected result | net lines |
|---|---|---|---|---|---|---|
| R1 | two physical-constant headers: `src/constants.h` (Rgas, KBoltz, Avogadro [1/mol], OneAtm, ElectronCharge; 16 includers) and `src/utils/constants.hpp` (a Cantera block: Avogadro [1/kmol], Boltzmann, GasConstant = Avogadro·Boltzmann, Planck, Faraday, StefanBoltz, Tiny, Undef, …, plus `IDN/IPR/IEN`, `Composition`, `npos`). No file references any constant of the Cantera block (`grep -rnwE "GasConstant|Boltzmann|Faraday|Tiny|Undef|IDN|IPR|IEN"` finds no use outside the header; tests/test_convective_adjustment.cpp uses IDN/IPR but is a canoe test that kintera's CMake never builds). Its only two includers use `Composition` (also defined at src/reaction.hpp:12) and `npos` | src/utils/constants.hpp:1-91; src/reaction.cpp:6,41,50; src/utils/parse_comp_string.cpp:8,28,47,50,67; src/reaction.hpp:12 | src/constants.h and reaction.hpp's `using Composition` | delete src/utils/constants.hpp; the two includers include `<kintera/reaction.hpp>` (parse_comp_string.hpp already returns a `Composition`-shaped map) and write `std::string::npos` | bitwise (nothing numeric is referenced) | −91 +2 = −89 |
| R2 | the ten `species_*` globals defined at src/species.cpp:34-44 are redeclared `extern` in four places; src/species.hpp declares six of them and the .cpp files add the other four each | src/species.hpp:131-136 (6); src/thermo/thermo_options.cpp:18-26 (9); copies in Jiachen's src/photochem/photochem.cpp:11-18 (8) and src/photochem/kinetics_base_reader.cpp:25-33 (9) — handed to him | species.hpp, extended to all ten | the per-file extern blocks go | bitwise | mine −9 +4 = −5 (his two copies −17 more) |
| R3 | `ThermoXImpl::reset` and `ThermoYImpl::reset` are the same 96 lines except `mu` vs `inv_mu`, Y's `cv0`/`u0` buffers and one log string: nucleation default and check, `check_dimensions`, the uref/sref offset normalisation (mutates `options`), verbose prints, the stoichiometry-matrix fill and the nucleation product check | src/thermo/thermo_x.cpp:38-133; src/thermo/thermo_y.cpp:38-138 (`diff` of the two ranges with the class name masked: 19 differing lines) | one free function, e.g. `torch::Tensor init_thermo_state(ThermoOptions const&, char const* tag)` next to `populate_thermo` in species.cpp, returning the stoich tensor | both `reset()`s call it and keep only their own buffers (X: `mu`; Y: `inv_mu`, `cv0`, `u0`, `uv_partitionable`, plans, counter) | bitwise: same ops in the same order; the verbose text keeps its `[ThermoX]`/`[ThermoY]` tag through the argument | ≈ −70 |
| R4 | two `eval_entropy_R` overloads; the `SpeciesThermo` one (gas entropy only) is the first 47 lines of the `ThermoOptions` one verbatim, and nothing calls it: all four callers pass a `ThermoOptions` (`grep -rn "eval_entropy_R("`: extrapolate_ad.cpp:124,235, thermo.hpp:321,424; no Python binding) | src/thermo/eval_uhs.cpp:274-334 and 336-382; decl src/thermo/eval_uhs.hpp:33-35 | the `ThermoOptions` overload | the `SpeciesThermo` overload and its declaration are deleted | bitwise (dead code) | −61 −3 = −64 |
| R5 | `extrapolate_dlnp` and `extrapolate_dz` repeat three blocks line for line: the verbose initial-state print (92-105 ≡ 202-215), the rain-out entropy target (122-136 ≡ 233-247) and the verbose iteration print (144-161 ≡ 253-270) | src/thermo/extrapolate_ad.cpp:92-105,202-215; 122-136,233-247; 144-161,253-270 | three file-local helpers (`print_state`, `rainout_entropy_target`, `print_iter`) in extrapolate_ad.cpp | both functions call them | bitwise: the rain-out block is the same tensor expression; prints are prints | −47 +15 = −32 |
| R6 | the "find the vapour and cloud of reaction j" loop (vapor_index/cloud_index/reactant_coefficient/product_coefficient) is written three times in the UV solver | src/thermo/equilibrate_uv.h:36-45, 89-98, 136-145 | one `DISPATCH_MACRO inline void reaction_pair(T const* stoich, int nspecies, int nreaction, int j, int& vapor, int& cloud, T& rc, T& pc)` at the top of the header | the three loops call it | bitwise (integer bookkeeping and sign tests only) | −30 +12 = −18 |
| R7 | the saturation-function registry is spelled three times: the CPU pointer table, the name vector (both src/func_table.cpp) and the CUDA table with a leading `nullptr` (src/func_table.cu); the func2 (h2 rotation) registry likewise | src/func_table.cpp:11-37; src/func_table.cu:10-50 | one X-macro list, e.g. `src/vapors/func_list.h` (`#define KINTERA_FUNC1_LIST(X) X(h2o_ideal) X(h2o_ideal_ddT) …`) | both .cpp and .cu expand the list for their table and the name vector | bitwise (same function pointers in the same order; `fill_func_table_*` looks names up, so the CUDA leading `nullptr` is kept by the macro) | ≈ −50 |
| R8 | `call_func1/2/3_cpu` and `call_func1/2/3_cuda` differ only in arity (one `reinterpret_cast` line and one call per extra argument) | src/utils/utils_dispatch.cpp:22-84; src/utils/utils_dispatch.cu:16-75 | one arity template per backend (`template <int N> call_funcN_cpu`), instantiated for 1, 2, 3 | the six functions become two templates plus six one-line wrappers for `REGISTER_*_DISPATCH` | bitwise (the per-element `out[j] += f(args…)` is unchanged) | ≈ −40 (cpp) −35 (cu) |
| R9 | the per-species push of cref_R/uref_R/sref_R/nasa9_*/names/mu is written once for vapours and once for clouds | src/thermo/thermo_options.cpp:218-227, 244-253 | one lambda `append_species(id)` | the second loop body is the lambda call | bitwise | −8 |
| R10 | the "reaction references unknown component" check is written for reactants and again for products | src/equilibrium/equilibrium_options.cpp:88-99 | one lambda over `Composition const&` | called twice | bitwise | −4 |
| R11 | NASA-9 h/RT and s/R are spelled twice with different expression trees: `nasa9_gibbs_RT` (invT2, `a3*T/2.0`, …) and eval_uhs's `nasa9_h_R`/`nasa9_s_R` (`T.pow(-2)`, `T.pow(2)/2`, …) | src/thermo/nasa9.hpp:22-66 (used by src/species.cpp:113 and src/kinetics/kinetics.cpp); src/thermo/eval_uhs.cpp:24-43 | eval_uhs's three functions (they also give cp/R) | `nasa9_gibbs_RT` = `nasa9_h_R − nasa9_s_R` | **not bitwise**: `log_k` of NASA-9 reactions and `nasa9_gibbs_rt` move at round-off (I expect ≤ 1e-15 relative; every NASA-9 kinetics deck moves). Listed for lane X's decision, not counted | −20 if taken |
| R12 | small shared fragments between the TP and UV device solvers: the "neither direction feasible → drop the reaction" swap (equilibrate_tp.h:175-180 ≡ equilibrate_uv.h:450-455) and the θ/λ step-limit loops (tp.h:285-292 ≡ uv.h:530-537) | src/thermo/equilibrate_tp.h:175-180, 285-292; src/thermo/equilibrate_uv.h:450-455, 530-537 | not proposed: 8-line fragments inside two 400-600-line templates; a helper would add more than it removes | — | — | 0 (noted only) |

Not one place but worth lane X's eye: `constants::Rgas = 8.31446` (src/constants.h:6) is a 6-digit truncation of Avogadro·KBoltz = 8.31446261815324 (the exact-SI value pyharp's Python side and snapy use elsewhere); the Python binding exposes it (python/csrc/pyconstants.cpp:13). Changing it moves every thermodynamic result by 3.1e-7 relative, so it is a release decision, not a refactor row.

## Question 2: what else can be cut

| id | what | sites file:line@26e359ed | which stays | others become | expected result | net lines |
|---|---|---|---|---|---|---|
| C1 | `src/thermo/thermo_prop.hpp`: included by nothing (`grep -rn thermo_prop.hpp src python tests` → none) and includes `<kintera/utils/func1.hpp>`, a header that does not exist, so it cannot compile; last touched by the nightly merge #80 | src/thermo/thermo_prop.hpp:1-39 | — | deleted | bitwise (never compiled) | −39 |
| C2 | `src/utils/units.h`: a second `class kintera::UnitSystem` (Cantera's, public `m_*` members) with no includer; it would violate the ODR against src/units/units.hpp:92 if ever included | src/utils/units.h:1-52 | src/units/units.hpp | deleted | bitwise (never compiled) | −52 |
| C3 | `src/utils/permute_matrix.cpp`: `permute_matrix_inplace`, compiled by the `utils/*.cpp` glob, no header, no caller anywhere (src, python, tests, examples) | src/utils/permute_matrix.cpp:1-51 | — | deleted | bitwise | −51 |
| C4 | `to_lower_copy`: no caller | src/utils/to_lower_copy.cpp:1-16, src/utils/to_lower_copy.hpp:1-10 | — | deleted | bitwise | −26 |
| C5 | `UnitSystem::set_defaults` (+ the `initializer_list` constructor that is its only caller) and `UnitSystem::convert_to`: no caller outside src/units (`grep -rnE "set_defaults\(|convert_to\(|UnitSystem\s*\(\{"` → none); the kinetics readers use the default constructor and `convert_from` | src/units/units.cpp:301-339, 355-371; src/units/units.hpp:95, 98, 107-108 | `convert`, `convert_from`, `defaults` | deleted | bitwise | −56 −4 = −60 |
| C6 | `src/vapors/ammonia.dat` (59 lines) and `src/vapors/fit_ammonia.py` (18): a fitting script and its input sitting in a header-only directory; nothing references them (`grep -rn "ammonia.dat\|fit_ammonia"` → only the script itself), `package-data` only ships `data/*.dat` | src/vapors/ammonia.dat, src/vapors/fit_ammonia.py | — | moved to docs/ or tools/ (provenance of the nh3 fit) or deleted; Cheng's call | bitwise | −77 |
| C7 | `parse_reactions_yaml` (src/utils/parse_yaml.{cpp,hpp}) is library code whose only callers are two of its own tests (tests/test_yaml_parse.cpp:33, tests/test_kinetics_rates.cpp:41); no production or Python caller | src/utils/parse_yaml.cpp:1-104, src/utils/parse_yaml.hpp:1-21 | — | either cut with its two tests or kept as a tested utility; lane X's decision, not counted | bitwise either way | −125 if cut |
| C8 | `tests/test_convective_adjustment.cpp` is a canoe/athena test (`#include <athena/mesh/mesh.hpp>`, `air_parcel.hpp`) that kintera's tests/CMakeLists.txt never builds; it is the only user of `IDN/IPR` (R1) | tests/test_convective_adjustment.cpp | — | deleted (tests/ is outside my half; noted for whoever owns tests/) | bitwise | not counted |

Counted total (R1–R10, C1–C6, my half only, all expected bitwise): −720 lines (415 from question 1, 305 from question 2). With lane X's two decisions (R11, C7): −865.

Not audited in depth: the solver bodies of equilibrate_tp.h / equilibrate_uv.h / phase_equilibrate_tp.h beyond the shared fragments (they are three different algorithms: TP active-set, UV with the energy equation, multi-phase TP), src/utils/alloc.h, src/loops.cuh, src/thermo/h2_rotation.h, src/kintera_formatter.hpp, src/diffusion (107 lines, nothing duplicated inside it), python/ and tests/. Not run: no build, no ctest, no gate; the bitwise calls are predictions to be checked row by row on CPU and CUDA when a row lands.

## Appendix: commands and outputs (all in a checkout of 26e359ed)

```
$ git rev-parse HEAD
26e359edc1b28f8bb4f75b13855c663f6e14a6d7

$ for d in thermo equilibrium vapors utils units diffusion; do printf "%-12s %6d\n" $d $(find src/$d -type f | xargs cat | wc -l); done
thermo         4691
equilibrium     673
vapors          203
utils          1891
units           534
diffusion      107
$ cat src/CMakeLists.txt src/add_arg.h src/constants.h src/func_table.cpp src/func_table.cu src/kintera_formatter.hpp src/loops.cuh src/reaction.cpp src/reaction.hpp src/species.cpp src/species.hpp | wc -l
1444

# include graph (aam/kaudit/includes.py: every #include in src, python, tests, examples, matched by path suffix); headers with zero includers:
src/thermo/thermo_prop.hpp     0
src/utils/to_lower_copy.hpp    0
src/utils/units.h              0
src/utils/constants.hpp        2   src/reaction.cpp, src/utils/parse_comp_string.cpp
src/utils/parse_yaml.hpp       2   tests/test_kinetics_rates.cpp, tests/test_yaml_parse.cpp

$ ls src/utils/func1.hpp
ls: cannot access 'src/utils/func1.hpp': No such file or directory
$ grep -rn "func1.hpp" src
src/thermo/thermo_prop.hpp:7:#include <kintera/utils/func1.hpp>

$ grep -rnE "constants::[A-Za-z_]+" src python | sed -E 's/.*constants::([A-Za-z_]+).*/\1/' | sort | uniq -c
     41 Rgas      (thermo.hpp 7, thermo_y.cpp 9, equilibrate_uv.h 8, extrapolate_ad.cpp 7, evaporation.cpp 3, kinetics.cpp 2, thermo_x.cpp 1, relative_humidity.cpp 1, pyconstants.cpp 1, …)
      3 Avogadro  (units.cpp 2, pyconstants.cpp 1)
      1 ElectronCharge (units.cpp)
      1 OneAtm    (units.cpp)
# none of GasConstant, Boltzmann, Planck, Faraday, StefanBoltz, Tiny, SmallNumber, BigNumber, Undef, Pi, OneBar, logGasConstant
$ grep -rnwE "IDN|IPR|IEN" src python tests examples | grep -v src/utils/constants.hpp
tests/test_convective_adjustment.cpp:87:  ac[0].w[IPR] = 9.61664e+06;
tests/test_convective_adjustment.cpp:88:  ac[0].w[IDN] = 4929.37;
$ grep -n convective_adjustment tests/CMakeLists.txt
(none)

$ for fn in to_lower_copy permute_matrix parse_yaml; do printf "%-16s " $fn; grep -rlw "$fn" src python tests examples | grep -vE "src/utils/$fn\.(cpp|hpp)" | tr '\n' ' '; echo; done
to_lower_copy
permute_matrix
parse_yaml       tests/test_kinetics_rates.cpp tests/test_yaml_parse.cpp
$ grep -rn "ammonia.dat\|fit_ammonia" .   (build dirs excluded)
src/vapors/fit_ammonia.py:5:data = genfromtxt('ammonia.dat')

$ grep -rn "eval_entropy_R(" src python tests | grep -v src/thermo/eval_uhs
src/thermo/extrapolate_ad.cpp:124:      auto scond = eval_entropy_R(temp, pres, conc_full, stoich, options)
src/thermo/extrapolate_ad.cpp:235:      auto scond = eval_entropy_R(temp, pres, conc_full, stoich, options)
src/thermo/thermo.hpp:321:        eval_entropy_R(temp, pres, conc, stoich, options) * constants::Rgas;
src/thermo/thermo.hpp:424:        eval_entropy_R(temp, pres, conc, stoich, options) * constants::Rgas;
# `options` is a ThermoOptions at all four sites

$ diff <(sed -n 38,133p src/thermo/thermo_x.cpp | sed 's/ThermoX/ThermoN/') <(sed -n 38,138p src/thermo/thermo_y.cpp | sed 's/ThermoY/ThermoN/') | grep -c '^[<>]'
19
# the 19 lines: X's `mu` buffer vs Y's `inv_mu`; Y's cv0/u0 block (9 lines); one log string; X's trailing verbose print

# identical ≥6-line blocks (aam/kaudit/dupblocks.py, stripped lines, window 6, merged runs), my half:
  16 lines  src/thermo/extrapolate_ad.cpp:90-100  ==  src/thermo/extrapolate_ad.cpp:200-210
  15 lines  src/thermo/thermo_y.cpp:105-114  ==  src/thermo/thermo_x.cpp:95-104
  14 lines  src/thermo/eval_uhs.cpp:280-288  ==  src/thermo/eval_uhs.cpp:342-350
  14 lines  src/thermo/equilibrate_uv.h:90-98  ==  src/thermo/equilibrate_uv.h:137-145
  13 lines  src/thermo/equilibrate_uv.h:38-45  ==  src/thermo/equilibrate_uv.h:138-145
  12 lines  src/thermo/eval_uhs.cpp:300-306  ==  src/thermo/eval_uhs.cpp:362-368
  11 lines  src/thermo/thermo_y.cpp:123-128  ==  src/thermo/thermo_x.cpp:113-118
  11 lines  src/thermo/extrapolate_ad.cpp:143-148  ==  src/thermo/extrapolate_ad.cpp:252-257
  11 lines  src/thermo/extrapolate_ad.cpp:121-126  ==  src/thermo/extrapolate_ad.cpp:232-237
   9 lines  src/thermo/thermo_options.cpp:218-221  ==  src/thermo/thermo_options.cpp:244-247
   8 lines  src/thermo/thermo_options.cpp:20-22  ==  src/photochem/photochem.cpp:12-14 (and kinetics_options.cpp:13-15)
   8 lines  src/thermo/equilibrate_uv.h:534-536  ==  src/thermo/equilibrate_tp.h:288-290
   8 lines  src/thermo/equilibrate_uv.h:449-451  ==  src/thermo/equilibrate_tp.h:175-177
   7 lines  src/utils/utils_dispatch.cpp:26-27  ==  :45-46  ==  :66-67
   8 lines  src/species.hpp:131-133  ==  src/photochem/kinetics_base_reader.cpp:25-27

$ grep -rnE "set_defaults\(|convert_to\(|UnitSystem\s*\(\{" src python tests examples | grep -v "^src/units/"
(none)
$ grep -n "extern std::vector\|extern bool" src/thermo/thermo_options.cpp src/photochem/photochem.cpp src/photochem/kinetics_base_reader.cpp src/species.hpp | awk -F: '{print $1}' | uniq -c
      9 src/thermo/thermo_options.cpp
      8 src/photochem/photochem.cpp
      9 src/photochem/kinetics_base_reader.cpp
      6 src/species.hpp

$ sed -n 86p pyproject.toml; git tag --sort=-v:refname | head -1
fallback_version = "2.5.13"
v2.6.11
# test_version_fallback at 26e359ed (from the K5 x86 leg):
AssertionError: a tree without git metadata reports 2.5.13+g26e359edc1, older than the release 2.6.11 it was exported from
```
