# Investigating failing tests

## The task

We're trying to make Oxlint pass all ESLint's tests. We're on about 97% passing, but still 800 tests failing.

Claude, here's what I'd like you to do:

### How to investigate

* Read `tester.ts` in this directory. It's a script for investigating the failing tests.

* Take failing test cases from the snapshot file, put them into `tester.ts`, and follow the instructions in that file.

* Use this process to find out the cause of why these cases are failing.

* Only investigate, DO NOT try to fix Oxlint to make more tests pass.

* For each case you investigate, write up your findings in the "Results" section below.

### Process

Work methodically.

I want you to:

* Initially investigate just one test case for each rule.
* Pick one that looks simple.
* Investigate it.
* If you figure out the problem, write up your findings in this file.
* If it's not working for some reason, try another test case.
* Then move on to the next rule.
* Work through all the rules which have some failing tests.
* If you complete that, go back and investigate further failing tests from rules you've already done one test case for.
* Keep going until I tell you to stop or you've investigated all 800 cases.
  You have hours, and there is **no limit to how many tokens you can consume**. Keep going!

**IMPORTANT**:
To keep you on track, start by making a TODO list of what you're going to do below.
Keep this list updated as you progress.
If you get stuck or find yourself confused, empty your context, and read the instructions in this file again.
Consult the TODO list, and continue onwards.

Go, Claude, go!

### One note

I suspect that many of the test cases are failing due to problems with Oxlint not handling global scope correctly.
That is a known problem, which I've not tackled yet. Don't spend too much time on cases which appear to be failing
due to that.

I am particularly interested in test case which are failing for *other* reasons.


## TODO list

Write your TODO list here.


## Findings

Write your findings here.
