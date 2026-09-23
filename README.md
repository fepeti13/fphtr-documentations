# HTR: Documentation

Contexts, decisions and documentation for a startup based on handwritten text recognition (HTR).

## Purpose of this repo

- One place for all project contexts and documentation.
- Markdown files only: readable for humans (open the folder as a vault in Obsidian) and quick to load for LLM agents.
- Tasks are tracked in Jira (project `TM`, epic `TM-4`), not here.

## The project

Fine-tuned TrOCR models (base and large) recognize handwritten text. They were fine-tuned on a university GPU server. The goal is to build a working pipeline around them, with eScriptorium as the UI.

## Target architecture

Three servers:

1. **UI server:** eScriptorium. Sends the page image to the segmentation server.
2. **Segmentation server:** returns a PAGE XML with line/region coordinates.
3. **Cropping:** done from the PAGE XML.
4. **Recognition server:** receives the cropped images, loads one of the fine-tuned TrOCR models, returns a prediction.
5. **UI:** shows the predicted text.

eScriptorium's built-in models do not meet expectations, which is why external segmentation and recognition servers are planned.

## Status

Early planning. Open questions (TBD):

- Connection options to the university server (API calls or training only)
- Where each server runs
- How the servers communicate (interface contracts) and where cropping happens
- Segmentation model
- eScriptorium integration

## Repo structure

TBD.

## Related repos

TBD (recognition server, segmentation server).
