# NOMAD: Network for Open Mobility Analysis and Data

Welcome to the NOMAD documentation site. NOMAD is an open-source platform designed for scalable analysis of GPS trajectory data and mobility metrics computation. It supports end-to-end processing: from ingestion and quality control to stop detection, metric computation, and synthetic trajectory generation. 

All functions are implemented in Python with parallel equivalents in PySpark, allowing seamless execution in both local and distributed environments.

---

## What NOMAD Offers

- Ingestion and validation of GPS data (CSV, Parquet, GeoJSON)
- Filtering and completeness measurement across time and space
- Tessellation using H3, S2, or custom spatial grids
- Stop and trip detection, including Project Lachesis
- Inference of home and workplace locations
- Computation of mobility metrics (e.g., entropy, radius of gyration)
- Co-location and contact network generation
- POI attribution and clustering
- Differentially private aggregation and debiasing
- Synthetic trajectory simulation using stochastic models

NOMAD builds on existing libraries like *scikit-mobility*, *mobilkit*, and *trackintel*, aiming to unify and extend these capabilities in a production-ready library for large-scale research.

---

## Documentation Overview

Use the sidebar or the links below to get started with different parts of the NOMAD pipeline:

- [Setup Instructions](setup.md): Install the NOMAD library and check your environment
- [Reading GPS Data](reading-data.md): Load and explore raw mobility datasets
- [Filtering and Pre-Processing](filtering.md): Clean and filter data by quality, time, and geography
- [Stop Detection in Trajectories](stop-detection.md): Apply rule-based and clustering-based detection methods

More tutorials and advanced modules will be added soon.

---


## Examples and Usage

Example notebooks demonstrating data loading, filtering, stop detection, and trajectory simulation are available in the `examples/` folder of the [GitHub repository](https://github.com/Watts-Lab/nomad). These notebooks are compatible with both local Python and Spark environments.

---

## License and Citation

NOMAD is developed by the University of Pennsylvania and released under the MIT License.

For more information, visit the main project site: [https://nomad.seas.upenn.edu](https://nomad.seas.upenn.edu).
