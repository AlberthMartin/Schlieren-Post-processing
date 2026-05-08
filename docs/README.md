# Schlieren Post-processing Docs

This directory documents how the repository works, with `main_weighted.py` as
the current main entry point.

Start here:

- [Main weighted pipeline](main-weighted-pipeline.md): step-by-step execution
  flow from file selection to CSV/video output.
- [Inputs, interaction, and outputs](usage-inputs-outputs.md): required files,
  runtime UI prompts, saved state, and generated artifacts.
- [Module reference](module-reference.md): what each Python file contributes to
  the processing pipeline.
- [Tuning and known caveats](tuning-and-caveats.md): the most important
  constants, switches, and current limitations.

## One-paragraph overview

The repository processes Phantom `.cine` schlieren/shadowgraph videos of sprays.
`main_weighted.py` loads one or more videos, converts them to 8-bit grayscale,
rotates the image so the spray points mostly to the right, asks for or reuses a
spray origin, crops a horizontal strip around that origin, enhances contrast,
computes optical-flow motion, combines intensity/motion/freehand/cone evidence
into a binary spray mask, extracts penetration and cone-angle metrics from the
mask boundary, displays diagnostic windows, saves an overlay video, asks for one
manual validation mask, plots metrics, and writes a per-frame CSV.

## Quick Run

Install dependencies:

```bash
pip install -r requirements.txt
```

Run the current pipeline:

```bash
python main_weighted.py
```

The script opens native file dialogs and OpenCV windows, so it must be run from
an environment with GUI display support.

