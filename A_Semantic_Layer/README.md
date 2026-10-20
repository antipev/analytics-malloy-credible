# MASTER README: MALLOY SEMANTIC MODEL
# Architecture: Layered Structure (Sources -> Extended Sources -> Models)

## 1. PROJECT OVERVIEW
This project implements a modular, scalable Malloy modeling structure. By separating raw data connections from business logic and relationship definitions, we ensure the model is resilient to upstream schema changes in BigQuery.

---

## 2. THE 4-LAYER ARCHITECTURE

### LAYER 0: SOURCES (Base)
**Location:** `/0_sources/`
**Purpose:** Direct mapping to BigQuery.
- **Rules:** No business logic, no measures, no joins.
- **Key File:** `config.malloy` (Defines `bigquery.table()` pointers).
- **Benefit:** If a table name changes in the warehouse, you only update it here once.

### LAYER 1: EXTENDED SOURCES (Staging)
**Location:** `/1_extended_sources/`
**Purpose:** Refining raw data into usable analytical entities.
- **Rules:** Every file imports from `/0_sources/` and uses the `extend` keyword.
- **Actions:** Define `primary_key`, rename technical columns to human-readable labels, and add basic dimensions.
- **Benefit:** Creates a "clean" version of the source that is consistent across all models.

### LAYER 2: MODEL (Curated)
**Location:** `/2_models/`
**Purpose:** Defining relationships, joins, and "Explores."
- **Rules:** Imports refined files from `/1_extended_sources/`.
- **Actions:** Define `join_one` or `join_many` relationships and complex cross-source measures.
- **Benefit:** This is the "End Product" for users to run queries against.

### LAYER 3: ANALYSIS (Output)
**Location:** `/3_analysis/`
**Purpose:** Final consumption layer for queries, reports, and dashboards.
- **Subfolders:**
    - `adhoc/`: Rapid, one-off queries and experimental analysis.
    - `dashboards/`: Production-ready dashboard definitions and visualizations.
    - `use_cases/`: Deep-dive analysis focused on specific business problems.

---

## 3. DIRECTORY STRUCTURE
/
├── 0_sources/             # Raw BigQuery table pointers
│   └── config.malloy
├── 1_extended_sources/    # Column renaming, PKs, and extensions
│   ├── orders.malloy
│   ├── users.malloy
│   └── ...                # Additional extended source files
├── 2_models/              # Relationship definitions & joins (Explores)
│   └── ecommerce_model.malloy
├── 3_analysis/            # Consumption and reporting layer
│   ├── adhoc/             # Quick scratchpad queries
│   ├── dashboards/        # Formal Malloy dashboard files
│   └── use_cases/         # Specialized business analysis
└── README.md              # Project Documentation

---

## 4. DEVELOPMENT WORKFLOW
1. **Define Source:** Add the raw table pointer to `0_sources/config.malloy`.
2. **Extend Source:** Create a file in `1_extended_sources/` that imports the config and uses `source: my_source is base_source extend { ... }`.
3. **Build Model:** Import the extended source into a model file in `2_models/` to define joins and complex metrics.
4. **Analyze:** Create query files in `3_analysis/` importing the models to produce final insights.

---

## 5. TECHNICAL REQUIREMENTS
- **VS Code:** With the official Malloy Extension.
- **Authentication:** Google Cloud SDK / BigQuery Authentication.
- **Connections:** The connection name in `0_sources/config.malloy` must match your local VS Code Malloy connection settings.