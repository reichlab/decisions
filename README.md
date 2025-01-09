# decisions

A centralized public record of decisions made for lab projects. There are three
kinds of decision documents in this repository: 

1. project posters are living documents that provide broad context for a
   project which may have one or more decisions associated with it,
2. lightweight decision records (LDR) are formal records of decisions that do
   not need explicit feedback, and
3. request for comments (RFC) are formal records of decisions that require
   feedback from the broader team

**For background and details about how decisions and project posters are
structured, see
[decisions/2024-12-09-rfc-record-decisions.md](decisions/2024-12-09-rfc-record-decisions.md)**.

## Proposing a Decision (LDR or RFC)

To propose a decision, follow these steps

1. create a new branch with your initials, the type of decision, and a short
   description of the decision (e.g. `ak/rfc/target-data`)
2. make a copy of
   [templates/decision-template.md](templates/decision-template.md) into the
   `decisions/` folder and name it with
   `YYYY-MM-DD-<project>-<decision_title>.md` where `<project>` is the name of
   the associated project and `<decision_title>` is a short description.
3. follow the instructions in the template
4. create a pull request with the format `[TYPE] decision title` where `[TYPE]`
   is either `[RFC]` or `[LDR]`
5. In the main comment for the pull request, specify the timeframe you believe is required for reviews
   (default: 3 days for LDR, 1 week for RFC)

Any supporting documents that are required for your decision should be placed
in a folder under `decisions/` with the same name as your decision document.
Note that files within the specific subfolder need not follow a specific naming strategy
but should have descriptive names.

## Creating a Project Poster

New projects require a project poster. To create a project poster,


1. create a new branch with your initials, the word "poster", and the project
   name (e.g. (znk/poster/hub-docker-containers)
2. make a copy of [templates/poster-template.md](templates/poster-template.md)
   into the a subfolder `posters/` with the format `<project>/<project>.md`
   where `<project>` is the name of the project. You can place any supporting
   files inside the `posters/<project>/` folder.
3. follow the instructions in the template
4. create a pull request with the format `[poster] project title`
5. request a review from your collaborators with a timeline (suggestion of 1 week)

## Review Process

Reviews are based on [Martha's Rules consensus
model](https://third-bit.com/2019/06/13/marthas-rules/) and adapted for an
asynchronous context. In the parlance of GitHub Pull Requests, the **sponsor**
is the person who opened the pull request and a pull request review is a
"sense" vote, where an approval indicates you like the proposal, a "comment"
indicates you can live with the proposal (though you may have some
reservations), and a "request changes" indicates that you are uncomfortable
with the proposal.

The process for approval or request for rework is detailed below for each type
of document.

### Project Posters

A quorum for any project poster will consist of 2/3 of the project members. It
is approved if the "request for changes" are not in the majority.

### Lightweight Decision Records

A quorum for any lightweight decision record will consist of 2/3 of the project
members. It is approved if the "request for changes" are not in the majority.

### Request for Comment

A quorum for any request for comment will consist of 2/3 of the lab members. If
there are any "request for changes" that cannot be resolved in two rounds of
review, it is sent for synchronous discussion at the week's development team
meeting. A discussion about the objections will be set for 10 minutes and then
a vote will be made. If it is approved, the RFC is merged, if it is not, the
RFC will be closed and the sponsor will rework the proposal and open a new RFC.

