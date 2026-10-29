# tenor

**tenor** is a hybrid English formality classifier for flagging informal sentences in learner writing. Three systems (a feature model, an LLM zero-shot scorer, and a hybrid that routes uncertain cases to the LLM) are evaluated on the same held-out set against one optimization metric, with bootstrap confidence intervals, a paired comparison, and a cost and latency table.

The name is Halliday's: tenor is the register dimension governed by who is speaking to whom, which is what formality encodes.

The design document was written before any code: [docs/design.md](docs/design.md).

Data: Pavlick and Tetreault (2016), via the Hugging Face mirror `osyvokon/pavlick-formality-scores`. See the design doc for citation and license notes.

Status: design phase. No results yet.
