# keyword generics initiative
<!--

 This is the template for creating an initiative in rust-lang. Be sure to go
 through all sections marked with `**FIX ME**`, and make sure that the text is
 correct, and feel free to replace/remove any part that's not relevant to
 your group.

 Steps to customize:

 * Edit CHARTER.md
 * Replace placeholder text (see below)
 * Remove references to "expermental" and "evaluation" unless you need them
 
  All of the text across all of the initial files uses the same group of
 variables to allow for easy search and replace. They are listed below.

 Example sed command: `sed -i '' 's/keyword generics/Inline ASM/g' ./**/*.md`
 *Note* you need `-i ''` on macOS and just `-i` on Linux.

 * keyword generics -> The display name of your group e.g. "Inline ASM".
 * keyword-generics-initiative -> The url slug name of your group used for
   `rust-lang/team` and repo name. e.g. "pg-inline-asm".
 * Zulip -> The name of your chat app e.g. "Zulip".
 * {{CHAT_LINK}} -> The hyperlink to your discussions on the chat app
   e.g. "https://rust-lang.zulipchat.com/#narrow/stream/216763-project-inline-asm".

To get your repo under rust-lang, file an infra issue:
https://github.com/rust-lang/infra-team/issues/new

-->

![initiative status: active](https://img.shields.io/badge/status-active-brightgreen.svg)

## What is this?

This page tracks the work of the keyword generics [initiative]! To learn more
about what we are trying to do, and to find out the people who are doing it,
take a look at the [charter]. 

[charter]: ./CHARTER.md
[initiative]: https://lang-team.rust-lang.org/initiatives.html

## Current status

The following table lists of the stages of an initiative, along with links to the artifacts that will be produced during that stage.

| Stage              | State | Artifact(s)                                                         |
| ------------------ | ----- | ------------------------------------------------------------------- |
| [Proposal]         | ✅    | [Proposal issue](https://github.com/rust-lang/lang-team/issues/162) |
|                    | ✅    | [Charter](./CHARTER.md)                                             |
| [Experimental]     | ✅    | [Evaluation](./evaluation/)                                         |
| [Development]      | 🦀   | [Draft RFCs](./explainer/)                                           |
|                    | 🦀   | [Tracking issue](https://github.com/rust-lang/rust/)                |
| [Feature complete] | 💤   | Stabilization report                                                |
| [Stabilized]       | 💤   |                                                                     |

[Proposal]: https://lang-team.rust-lang.org/initiatives/process/stages/proposal.html
[Experimental]: https://lang-team.rust-lang.org/initiatives/process/stages/proposal.html
[Development]: https://lang-team.rust-lang.org/initiatives/process/stages/development.html
[Feature complete]: https://lang-team.rust-lang.org/initiatives/process/stages/feature-complete.html
[Stabilized]: https://lang-team.rust-lang.org/initiatives/process/stages/stabilized.html

Key:

* ✅ -- phase complete
* 🦀 -- phase in progress
* 💤 -- phase not started yet

## How Can I Get Involved?

* Check for 'help wanted' issues on this repository!
* If you would like to help with development, please contact the [owner](./charter.md#membership) to find out if there are things that need doing.
* If you would like to help with the design, check the list of active [design questions](./design-questions/README.md) first. 
* If you have questions about the design, you can file an issue, but be sure to check the [FAQ](./FAQ.md) or the [design-questions](./design-questions/README.md) first to see if there is already something that covers your topic.
* If you are using the feature and would like to provide feedback about your experiences, please [open a "experience report" issue].
* If you are using the feature and would like to report a bug, please open a regular issue.

We also participate on [Zulip][chat-link], feel free to introduce yourself over there and ask us any questions you have.

[open issues]: /issues
[chat-link]: https://rust-lang.zulipchat.com/#narrow/stream/328082-t-lang.2Fkeyword-generics
[team-toml]: https://github.com/rust-lang/team/blob/master/teams/initiative-keyword-generics-initiative.toml

## Building Documentation
This repository is also an mdbook project. You can view and build it using the
following command.

```
mdbook serve
```
