# Rod Enhancement Proposals (REPs)

A Rod Enhancement Proposal (REP) is a way to propose, communicate and coordinate on new efforts for the Rod project. You can read the full details of the project in [REP-0](./0-rep).

## Quick start

1. Understand what component your REP is changing
2. Copy `NNNN-rep-template` folder to your component folder and
   1. Change `NNNN` to the next REP number (`find proposals/ -type d | egrep '[0-9]+-' | sed 's/^.*\///g' | sort`)
   2. Change `rep-template` to the title of your change
3. Adjust `README.md` and `rep.yaml`


## Why REP is needed

REPs are required for most non-trivial changes. Specifically:

* Anything that may be controversial
* Major changes to existing features
* Changes that are wide ranging or impact most of the project
