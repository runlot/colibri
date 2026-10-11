# Contributing

Colibri is licensed under the Colibri Source License 1.0 (`LICENSE`), with a
Change Date of four years to Apache-2.0. Because Runlot also offers the software
under a commercial licence, every contribution needs a **Contributor License
Agreement**: you keep your copyright and grant Runlot the right to distribute
your contribution under the project's licences, including the commercial one.
A DCO sign-off is not sufficient for this project.

The CLA is signed once, through the bot that comments on your first pull
request. Until it is signed the pull request is not merged.

Everything in this repository is written in English: code, comments, commit
messages, documents, log lines.

Dependencies must be under a licence in `deny.toml`'s allow list; CI runs
`cargo deny check licenses` and a copyleft dependency fails the build.

Tests are split (design D18): the harnesses and the unit and integration tests are in
this repository; the scenario corpus and regression seeds are not, and CI runs both. If
your pull request fails only there, a maintainer will reply with a reduced reproduction
you can run here.
