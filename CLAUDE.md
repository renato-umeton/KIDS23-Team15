# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

VGenes is a PyQt5-based graphical platform for multimodal analysis of B cell receptor (BCR) repertoire data. It integrates BCR sequencing with transcriptome expression, surface protein expression, and antigen probe binding data from single-cell sequencing. The application enables antibody candidate selection and clonal analysis for immunology research.

## Application Architecture

### Core Application Structure

- **VGenesMain.py**: Main application entry point (1,600+ lines). Contains the primary QMainWindow class with UI logic, signal handlers, and orchestration of all modules.
- **ui_VGenesMain.py**: Auto-generated PyQt5 UI code from VGenesMain.ui (do not edit manually).
- **VGenesMain.ui**: Qt Designer file defining the main window layout and widgets.

### Key Modules

**Database Layer:**
- **VGenesSQL.py**: SQLite database operations. Creates and manages the BCR database schema with fields for V/D/J gene assignments, CDR regions, mutations, and metadata.
- Database schema: Main table `vgenesdb` with ~100+ fields tracking gene assignments, framework regions (FR1-3), CDR regions (CDR1-3), mutations, quality metrics, and user annotations.
- Metadata table: `fieldsname` stores field definitions, nicknames, types, and display properties.

**Sequence Analysis:**
- **IgBLASTer.py**: Interface to NCBI IgBLAST for V(D)J gene assignment. Manages subprocess calls to `igblastn` binary in IgBlast/ directory.
- **VGenesSeq.py**: Sequence manipulation utilities including translation, codon handling, isoelectric point calculation, and molecular weight calculations.
- **VGenesCloneCaller.py**: Clonal grouping algorithm that clusters sequences by V/J gene usage and CDR3 properties.

**Reporting and Visualization:**
- **VReports.py**: Report generation module with export capabilities.
- **VMapHotspots.py**: Somatic hypermutation (SHM) hotspot mapping and visualization.
- **Plotting**: Integrated with matplotlib, seaborn, pyqtgraph, and pyecharts for various chart types.

**UI Components:**
- 37 .ui files defining dialog boxes for various operations (import, export, analysis, visualization).
- Corresponding ui_*.py files are auto-generated using pyuic5.
- **VgenesTextEdit.py**: Custom text editor widget for sequence editing.
- **VGenesDialogues.py**: Common dialog utilities (file open/save, messages).

**External Tools Integration:**
- **Tools/**: Contains bioinformatics binaries:
  - `clustalo`: Multiple sequence alignment (ClustalOmega)
  - `muscle`: Alternative MSA tool
  - `raxml`: Phylogenetic tree construction
  - `makeblastdb`: BLAST database creation
- **IgBlast/**: NCBI IgBLAST installation with germline databases in IG/ subdirectory.
- **Data/**: Reference databases (VDJGenes.db), HTML templates for visualization, connector sequences for cloning.

**Single-cell Integration:**
- **scRNAseq.py** / **scRNApage.py**: Integration with Scanpy for single-cell RNA-seq analysis alongside BCR data.

## Building and Running

### Application Packaging

Build standalone application using PyInstaller:

```bash
pyinstaller VGenesMain.spec
```

For Windows build, use:
```bash
pyinstaller VGenesMain-win.spec
```

The spec file includes all necessary data files (Js/, Data/, IgBlast/, Tools/, Conf/) and sets recursion limit to 5000.

### UI Development

When modifying UI files:

1. Edit .ui files in Qt Designer
2. Regenerate Python code:
```bash
pyuic5 VGenesMain.ui -o ui_VGenesMain.py
pyuic5 import_data_dialog.ui -o ui_import_data_dialog.py
# etc. for other dialog files
```

Do not manually edit ui_*.py files - changes will be overwritten.

### Running from Source

```bash
python VGenesMain.py
```

Dependencies include: PyQt5, sqlite3, pandas, numpy, biopython, scanpy, matplotlib, seaborn, pyqtgraph, pyecharts, weblogo, pillow

## Database Schema

The VGenes database (.vdb files, which are SQLite databases) stores:

- **Sequence identifiers**: SeqName (unique), SeqLen, GeneType
- **Gene assignments**: V1/V2/V3 (top 3 V gene hits), D1/D2/D3, J1/J2/J3
- **Productivity**: StopCodon, ReadingFrame, productive flag
- **Regions**: Framework regions (FR1/2/3) and CDR regions (CDR1/2/3) with positions and identity percentages
- **Junctions**: VDJunction, DJJunction, VJunction boundaries
- **CDR3**: CDR3DNA, CDR3AA, CDR3Length, CDR3beg/end, CDR3MW (molecular weight), CDR3pI (isoelectric point)
- **Germline**: GermlineSequence with germline positions (GVbeg/end, GD1beg/end, GJbeg/end)
- **Mutations**: TotalMuts, Mutations (list), SeqAlignment from IgBLAST
- **Metadata**: Project, Grouping, SubGroup, Species, Isotype, DateEntered, Comments, Quality
- **Clonality**: ClonalPool, ClonalRank (from clone calling)
- **Specificity**: Specificity, Subspecificity (user annotations)
- **User fields**: Blank6-Blank20 for custom annotations

Query using VGenesSQL module functions or direct sqlite3 connections.

## Key Workflows

### Sequence Import and Analysis

1. Import sequences via Import dialog (ui_import_data_dialog.ui)
2. IgBLASTer.py runs IgBLAST analysis on sequences
3. Parse IgBLAST output and populate database fields
4. Calculate CDR3 properties (MW, pI) using VGenesSeq utilities

### Clonal Grouping

VGenesCloneCaller.CloneCaller() groups sequences by:
- Identical V locus
- Identical J locus
- Same CDR3 length
- CDR3 sequence similarity
- Mutation patterns

Returns clonal pools with ranking.

### Visualization

Charts use multiple libraries:
- pyqtgraph: Fast interactive plots (custom PlotItem and ViewBox in PyQtGraphPlotItem.py, PyQtGraphViewBox.py)
- matplotlib: Static publication-quality plots
- pyecharts: Web-based interactive charts (HTML templates in Data/)
- QChart: Native Qt charts

Web visualizations use QWebEngineView to display HTML/JavaScript charts.

## Platform-Specific Code

Code handles Windows/macOS/Linux differences:

```python
from platform import system

if system() == 'Windows':
    igblast_path = os.path.join(working_prefix, 'IgBlast', 'igblastn.exe')
else:
    igblast_path = os.path.join(working_prefix, 'IgBlast', 'igblastn')
```

Tool binaries include both .exe (Windows) and native executables (macOS/Linux).

## Configuration and Resources

- **Conf/**: Configuration files including path_setting.txt and RecentPaths.vtx
- **Temp/**: Temporary files and error logs (ErLog.txt, ErLog2.txt)
- **Resources/**: Application resources compiled into VgenesResources_rc.py
- **Js/**: JavaScript libraries for web-based visualizations
- **chart/**: Chart-related resources
- **mdi/**: MDI (Multiple Document Interface) resources
- **qtweb-resources/**: Web view resources

## Important Notes

- Database files use .vdb extension but are standard SQLite databases
- Working directory is determined by sys.argv[0] and stored in global `working_prefix`
- Thread pools handle parallel IgBLAST processing
- QThread classes used for long-running operations to keep UI responsive
- Signal/slot architecture for communication between modules and UI
