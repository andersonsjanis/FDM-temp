# FDM-temp

A browser-based temperature simulation of FDM / FFF printed parts. FDM-temp implements the 3D transient thermal model **T4F3** published by Zhang, Van Hooreweder and Ferraris (2022).

The whole tool is a single HTML file. It runs in any current browser, needs no installation and no server, and works offline.

**Live version:** `https://<your-username>.github.io/fdm-temp/`

## Source and acknowledgement

This tool implements the model published in:

> Zhang, J., Van Hooreweder, B., & Ferraris, E. (2022). T4F3: temperature for fused filament fabrication. *Progress in Additive Manufacturing*, 7, 971-991. https://doi.org/10.1007/s40964-022-00271-0

The model, its equations, the boundary-condition rules and the layout of the original application are the work of the authors at the Department of Mechanical Engineering, KU Leuven. The original T4F3 application (Windows and MATLAB) is available from the KU Leuven Advanced Manufacturing Lab: https://iiw.kuleuven.be/onderzoek/aml/technologyoffer

The paper is published open access under a [Creative Commons Attribution 4.0 International License](http://creativecommons.org/licenses/by/4.0/).

FDM-temp is an independent implementation written from the equations in the paper. It contains no code from the original application, and it is not affiliated with or endorsed by the authors or KU Leuven. The name FDM-temp is used to keep it clearly separate from the original T4F3 software. If you use results from this tool, cite the original paper (BibTeX below).

## What it does

- Simulates the temperature of every element of a cuboid part (single wall, double wall or solid block) during printing and cooling.
- **Inputs:** the same inputs as the original T4F3 application:
  - geometry
  - density and specific heat, either constant or as temperature-dependent tables
  - thermal conductivity with optional anisotropy, and emissivity
  - nozzle, plate and room temperatures, and the air temperature above the plate Ta(z)
  - printing speed, convection coefficient, strand width and layer thickness
  - run length, convection after the print, and the number of elements along the length
- **Print view:** the part in 3D, coloured by temperature, with the nozzle, its travel direction and the remaining path of the current layer. A time slider and playback step through the print and the cooling.
- **Probes:** up to 5 elements. Probes chosen before a run are recorded at every time step.
- **Plots:**
  - temperature against time at the probes, with drag-to-zoom
  - temperature against height through each probe's column, with Ta(z) for reference
- **Running:** the simulation runs in a background thread. A progress bar shows percent done, layer, elapsed time, time left and speed, and warns if the solver stops responding. Cancel keeps the results computed so far.
- **Checks and output:** a stability check (Eq. 5), a data-size estimate, save/load of full results (`.fdmtemp`), and CSV export of probe curves and single time frames.

## Use

- **Online:** open the GitHub Pages link above.
- **Offline:** download `index.html` (or `FDM-temp.html`) and open it in a browser.

Saved data and CSV exports go to the browser's downloads folder.

### Temperature-dependent material tables

Plain text with two columns: temperature in °C and the value (density in g/cm³ or cp in J/g/K). Columns can be separated by spaces, tabs, commas or semicolons. Lines starting with `#` are ignored.

```
# T [C]   rho [g/cm3]
25        1.250
60        1.226
200       1.110
```

Values between table points use pchip interpolation, as in the paper (Fig. 15). Each element uses ρ(T)·cp(T) at its own temperature at every time step.

## Model summary

- **Hypotheses (paper, Section 2.1):**
  - ideal cuboid voxel elements deposited in sequence
  - the printed part does not move or deform
  - no heat from crystallisation
  - radiation only between the part and the far environment
- **Geometry:** elements of Δx × Δy × Δz.
  - Δy is the strand width and Δz the layer thickness.
  - Δx = L / NL, and the time step is Δt = Δx / v (one element is deposited per time step).
  - Each layer starts at x = 0 on the front strand. Strands alternate direction, and every layer repeats the same path with no travel time.
- **Solver:** the explicit 7-point finite-difference scheme (Eqs. 3-4). Each new element starts at the nozzle temperature.
- **Boundaries:** a missing neighbour (not yet printed, or outside the part) is replaced using the convection + radiation boundary condition (Eqs. 6-13). The choice between the O(Δ) and O(Δ²) forms follows the rules in the paper's Fig. 13.
  - Convection uses the air temperature Ta(z) (Eq. 7); radiation uses the room temperature T∞.
  - The plate is a fixed temperature Tp under the first layer.
- **Stability:** Δt/(ρ·cp) · (λx/Δx² + λy/Δy² + λz/Δz²) ≤ 0.5 (Eq. 5, anisotropic form).

## Implementation notes

These points are not fully specified in the paper and are interpreted here:

- **Ta(z) fields:** the three fields are (1) on/off for Eq. 7, (2) the characteristic length lc in mm (9.443 mm in the paper), and (3) the air temperature far above the plate.
- **How long to run the sim.:** a multiple of the print time t_end.
- **Natural conv. after finish?:** the convection coefficient used once the last element is deposited (pseudocode step 11).
- **Optimise for stability:** sets Δx = (Δy⁻² + Δz⁻²)^(−1/2), which minimises the Eq. 5 stability number. The paper prints the exponent as +1/2; −1/2 is the dimensionally correct form and gives the original application's default of 75 elements for an 18 mm part.
- **Fig. 13 followed literally:** this includes the single-wall case, where the left side takes the O(Δy²) form built on the right-side boundary value.
- **Energy conservation:** the O(Δ²) boundary form doubles the exchange with the opposite neighbour, so heat is not conserved where it is used (for example the top element of each layer). This is kept so that results match the original T4F3 model.
- **Wide flat nozzle tip:** listed as an input in the paper, but no equations are given. It is not implemented.

## Verification

- An independent Python implementation of the same scheme matches this tool to within 1e-5 °C, with both constant and temperature-dependent properties.
- Against the paper's Sets 1 and 2 (double wall, 18 × 0.8 × 12 mm, PLA), using constant ρ = 1.226 g/cm³ and cp = 1.801 J/g/K:

| Check | Paper | This tool |
|---|---|---|
| Deposition time at r = (9, 0, 12) mm, v = 10 mm/s | 141.30 s | 141.31 s |
| Deposition time at r = (9, 0, 6) mm, v = 5 / 20 mm/s | 138.60 / 34.65 s | 138.64 / 34.66 s |
| Max difference, h = 60 vs 8.5 W/m²/K | 26 °C | 26.1 °C |
| Max difference, ε = 0.78 vs 0 | 4.1 °C | 3.4 °C |
| Inter-layer reheat temperature at v = 60 mm/s | ~160 °C | ~157 °C |

The paper used temperature-dependent ρ and cp for these sets, which accounts for the remaining differences.

## Limitations

- Cuboid geometry and the fixed zigzag path only, as in the original model.
- Results are held in browser memory. For large runs, the full temperature field is stored every n-th step to stay under about 160 MB; probes chosen before the run are still recorded at every step.
- The explicit scheme needs the stability number ≤ 0.5. Lowering the speed makes it worse, because Δt = Δx / v grows.

## How to cite

If you use this tool, cite the original paper:

```bibtex
@article{zhang2022t4f3,
  author  = {Zhang, Jie and Van Hooreweder, Brecht and Ferraris, Eleonora},
  title   = {{T4F3}: temperature for fused filament fabrication},
  journal = {Progress in Additive Manufacturing},
  volume  = {7},
  pages   = {971--991},
  year    = {2022},
  doi     = {10.1007/s40964-022-00271-0}
}
```

## License

<!-- Choose a license for this re-implementation (for example MIT) and add a LICENSE file. The paper itself is CC BY 4.0. -->
