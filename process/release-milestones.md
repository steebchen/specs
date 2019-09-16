# Release Milestones

- Owner: @janpio
- Stakeholders: @schickling @sorenbs
- Spec State: Outdated 🚨

We use GitHub Milestones to plan our releases. We use the following process to collect, nominate and commit issues to these milestones, then implement and release it, and reflect on the milestone afterwards:

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->


- [Prerequisite](#prerequisite)
- [Candidate Nomination](#candidate-nomination)
- [Milestone Planning](#milestone-planning)
  - [Preparation](#preparation)
  - [Meeting](#meeting)
- [Milestone implementation work](#milestone-implementation-work)
  - [Unassigning](#unassigning)
  - [Closing](#closing)
- [Release](#release)
- [Reconciliation and Retrospective](#reconciliation-and-retrospective)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Prerequisite

triaged - only those are considered ready

## Candidate Nomination

Team members apply `process/candidate` label to _nominate_ issues for the release and them to be included in the upcoming milestone planning meeting

## Milestone Planning

### Preparation

- Create milestones in all relevant repos

### Meeting

- List all `process/candidate` issues
- Person that nominated explains, together we commit if it should be included (issue is added to milestone) or not
- The person that will start to drive the issue during the work phase gets assigned
- At the end of the meeting 
  - all remaining `process/candidate` labels get _dropped_ back to the normal issues: Copy the list of issues to some internal issue (just copy/paste a list of the issues into it as a comment) and remove the label. This way they can be specifically be reconsidered for next release during the nomination phase.
  - all milestone issues have to be assigned to someone to drive them

## Milestone implementation work

### Unassigning

If the current "driver" realizes that the issue is not actionable or does not fit into the available time, he unassigns the issue. If it is actually broken, it can also be closed

### Closing

If some code was merged into the codebases that solves a problem that was tracked via an issue, the issue is closed (which moves the milestone forward).

## Release

- closed issues are considered to be included in the release

## Reconciliation and Retrospective

meeting where we talk about issues that did not get closed but were part of the milestone, decide if to instantly re-nominate (apply `process/candidate` label) or move back to "ready" pool
