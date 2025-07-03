
<h1 align="left">
CatGrowing QGIS Plugin
  <img src="https://github.com/user-attachments/assets/e3e350f6-8f4c-4285-bc0c-7aa5522fe832" align="right" width="100"/>
</h1>




> **A region‑growing algorithm for categorical raster data**

[![License: GPL v3](https://img.shields.io/badge/License-GPLv3-blue.svg)](LICENSE)
[![QGIS 3.x](https://img.shields.io/badge/QGIS-3.x-green.svg)](https://qgis.org)

---

CatGrowing is a **QGIS Processing** plugin that implements a histogram‑based *region‑growing* algorithm.
Starting from user‑supplied vector *seed* polygons and a categorical raster, the tool expands each seed until the surrounding landscape diverges (according to the **Orloci Chord distance**) beyond a user‑defined threshold.

This tool is particularly useful for **delineating spatial regions of similar categorical composition**, such as ecological patches, land systems, or any application requiring cluster growth in categorical maps.

> **Reference implementation:** Estación Experimental de Zonas Áridas (EEZA‑CSIC) – Developed by **Alberto Ruiz‑Rancaño** & **Gabriel del Barrio Escribano**.
---

## Table of Contents

1. [Installation](#installation)
2. [Quick Start](#quickstart)
3. [Parameters](#parameters)
4. [Algorithm Workflow](#algorithmworkflow)
5. [Example](#example)
6. [Citation](#citation)
7. [Acknowledgments](#acknowledgments)
8. [License](#license)

---


## Installation

### Prerequisites
    QGIS version 3.30 or higher.

1. **Plugin Manager (recommended)**

   1. Open **QGIS ► Plugins ► Manage and Install Plugins…**
   2. Search for **“CatGrowing”** and click **Install**.

2. **Manual install (dev builds)**

   ```bash
   git clone https://github.com/Aruiz99/CatGrowing.git
   cp -r CatGrowing ~/.local/share/QGIS/QGIS3/profiles/default/python/plugins/
   # Restart QGIS and enable the plugin
   ```

---

## Quick Start

1. **Prepare data**  – Ensure your categorical raster uses integer categories.
2. **Load seeds**    – Add a polygon layer that marks the initial regions.
3. **Open Processing ► Toolbox ► CatGrowing ► Categorical region growing**.
4. **Set parameters**

   * *Input vector layer* – your seeds
   * *Input raster layer* – categorical base map
   * *Rasterize field* – unique ID for every seed polygon
   * *Kernel size* – controls neighbourhood window
   * *Threshold* – dissimilarity tolerance (`0 = strict`, `√2 = max`)
5. **Run** – Two new rasters appear in the layer list:

   * *Rasterised input* – seed IDs as raster
   * *Output* – grown regions

---

## Parameters

| Parameter              | Type                | Description                                                                              |
| ---------------------- | ------------------- | ---------------------------------------------------------------------------------------- |
| **Input vector layer** | `VectorAnyGeometry` | Input vector layer containing seed polygons.                                             |
| **Input raster layer** | `RasterLayer`       | Input categorical raster layer. Must be single‑band, *categorical* raster (GeoTIFF or any GDAL‑readable)                  |
| **Rasterize field**    | `Integer`           | Attribute used for vector rasterization. Must be numeric and provide unique value per seed.          |
| **Kernel size**        | `Integer (1–25)`    | Side of the square window (pixels) used to compute local histograms. (`3, 5, 7` are good kernel sizes).                  |
| **Threshold**          | `Double [0–√2]`     | Threshold value for Orloci criteria. Values near 0 allow only very similar pixels to be included (strict growth), while values near √2 accept very dissimilar areas (uncontrolled expansion). A value around 0.57 balances precision and flexibility, and is a good empirical starting point. |
| **Rasterised input**   | `RasterDestination` | Path to save the rasterized input layer.                                             |
| **Output**             | `RasterDestination` | Path to save the final grown regions layer.                               |

---

## Algorithm Workflow

The *CatGrowing* workflow can be understood as an **iterative, histogram‑based region‑growing routine**:

1. **Seed definition** – One or more polygons mark the *seed* regions from which growth starts.
2. **Baseline composition** – For every seed the plugin builds a **category histogram** of the underlying raster cells. This histogram remains *fixed* throughout the entire run; newly added pixels do **not** alter the reference composition.

   <!-- Salto adicional -->
   ![Image](https://github.com/user-attachments/assets/62c07221-ea5d-4c38-8b44-80e909b4b873)
   <!-- Salto adicional -->
   
4. **Kernel sampling along the frontier**

   * The current frontier (outer edge) of the region is identified via morphological dilation.
   * For each frontier pixel a square **kernel window** of user‑defined side length is extracted from the raster.
   * The category histogram inside that window is calculated.
     
     
   <!-- Salto adicional -->
   ![Image](https://github.com/user-attachments/assets/d8a1f12e-db7d-41a1-b0cb-728ff9ad31c7)
   <!-- Salto adicional -->
   
6. **Dissimilarity test** – The *sample's* histogram is compared with the *seed’s* baseline histogram using the **Orloci Chord distance**. If the distance `D` is below the chosen **threshold** the **centre pixel** of the kernel is marked as *similar* and queued for inclusion.
 
    > ### Orloci Chord Distance  
    > For two frequency vectors **f₁** and **f₂** of length *n*:
    >
    > $D = \sqrt{2\left(1 - \frac{\sum_{i=1}^{n} f_{i1} f_{i2}}{\sqrt{\left(\sum_{i=1}^{n} f_{i1}^{2}\right)\left(\sum_{i=1}^{n} f_{i2}^{2}\right)}}\right)}$
    >
    > * `D ∈ [0, √2]`; smaller values mean greater similarity.
    >
    > ---
    > **Reference**: Orlóci, L. (1967). *An agglomerative method for classification of plant communities*. *The Journal of Ecology*, 55(1), 193–206. [https://doi.org/10.2307/2257725](https://doi.org/10.2307/2257725)


8. **Growth** – After every frontier pixel has been evaluated the queued pixels are added to the region and steps 1 – 4 repeat with the updated frontier. The loop stops when **no additional pixels** satisfy the dissimilarity criterion.

Each seed is processed independently and results are merged into a final raster at the end of the run.



> **Why use a fixed seed histogram?**  Keeping the reference composition constant prevents *mode‑locking*. The algorithm does not dilute the seed’s identity as it grows, which leads to clearer region boundaries – especially useful when working with highly heterogeneous categorical rasters.

---


## Example

### Input Data

The plugin includes example data in the `example` folder of the repository. To test the plugin, follow these steps:

1. Navigate to the `example` folder in the cloned repository:

   ```
   example/
   ├── CategoricalRasterData/
   │   ├── PIE-CLC2018_V2020_1K.RDC
   │   ├── PIE-CLC2018_V2020_1K.RST
   │   └── PIE-CLC2018_V2020_1K.smp
   ├── VectorData/
   │   ├── RedNature2000_ExampleZones.dbf
   │   ├── RedNature2000_ExampleZones.prj
   │   ├── RedNature2000_ExampleZones.shp
   │   └── RedNature2000_ExampleZones.shx
   ```

2. Use the following files:

   * **Vector Layer**: `VectorData/RedNature2000_ExampleZones.shp` (contains seed polygons).
   * **Raster Layer**: `CategoricalRasterData/PIE-CLC2018_V2020_1K.RST` (categorical raster data).

### Parameters

Set the following parameters in the plugin dialog:

| Parameter          | Value                                                    |
| ------------------ | -------------------------------------------------------- |
| Input Vector Layer | `example/VectorData/RedNature2000_ExampleZones.shp`      |
| Input Raster Layer | `example/CategoricalRasterData/PIE-CLC2018_V2020_1K.RST` |
| Rasterize Field    | `OBJECT_ID`  |
| Kernel Size        | `3`                                                      |
| Threshold          | `0.57317317`                                             |

### Outputs

After running the plugin, the following output files will be generated:

| Output File                    | Description                                   |
| ------------------------------ | --------------------------------------------- |
| `example/rasterized_seeds.tif` | Rasterized version of the input vector layer. |
| `example/grown_regions.tif`    | Final raster representing the grown regions.  |

### Steps to Run

1. Open QGIS and load the plugin.
2. Select the input files from the `example` folder.
3. Configure the parameters as shown above.
4. Run the algorithm and verify the outputs in the `example` folder.

---


## Citation

If you use CatGrowing in scientific work, please cite:

> Ruiz‑Rancaño, A. & del Barrio Escribano, G. (2025).  *CatGrowing: A QGIS plugin for categorical region‑growing analysis*.  Consejo Superior de Investigaciones Científicas (CSIC), Estación Experimental de Zonas Áridas (EEZA).  -  [https://doi.org/10.5281/zenodo.15798474](https://doi.org/10.5281/zenodo.15798474)

[![DOI](https://zenodo.org/badge/DOI/10.5281/zenodo.15798474.svg)](https://doi.org/10.5281/zenodo.15798474)
---

## Acknowledgments

We would like to thank all partner institutions, experts, and contributors whose support and collaboration have made the **MedConecta** project possible. The development of the **CatGrowing** plugin was carried out as part of this project, with the aim of supporting research and analysis of ecological connectivity in Mediterranean arid regions.

Special thanks go to the data providers — including the **Subdirección de Espacios Naturales Protegidos** of the **Comunidad Valenciana** and **Andalucía**, the **Dirección General de Patrimonio Natural y Acción Climática**, and the **Consejería de Agua, Agricultura, Ganadería, Pesca, Medio Ambiente y Emergencia** of the **Region of Murcia** — for their valuable contributions.

**MedConecta** has been selected under the 2021 call for support to biodiversity management research programmes and projects. It is supported by **Fundación Biodiversidad** of the **Ministry for the Ecological Transition and the Demographic Challenge (MITECO)**, within the framework of the **Recovery, Transformation and Resilience Plan (PRTR)**, funded by the **European Union – NextGenerationEU**.

---

## License

CatGrowing is released under the **GNU General Public License v3.0** – see [`LICENSE`](LICENSE) for details.

---

© 2025 CSIC‑EEZA.  All trademarks are the property of their respective owners.
