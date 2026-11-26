# Environments — Library Platform

| Environment | Type | Used for | Data |
|---|---|---|---|
| Scratch org | Temporary, from source | Daily development and unit tests | Synthetic seed data |
| Dev sandbox | Developer or Partial Copy | Integration testing across packages and the Node API | Synthetic seed data |
| UAT / pilot sandbox | Full or Partial Copy | Librarian acceptance testing and final checks before release | Masked or synthetic data |
| Pilot library org | Production | First real library, with staged rollout | Real library data |
| Client orgs | Production | Each other library | Real library data |
