# git-history-manager

Estensione VS Code che riproduce il flusso di lavoro Git di IntelliJ IDEA.

## Branch

| Branch | Ruolo |
|---|---|
| `main` | Solo versioni rilasciate. Ogni merge su `main` ha un tag `vX.Y.Z`. |
| `develop` | Integrazione. Base di partenza per ogni nuovo lavoro. |
| `feature/<spec>-<descrizione>` | Lavoro su una specifica, ad esempio `feature/s0-git-runner`. Parte da `develop`, torna in `develop`. |
| `fix/<descrizione>` | Correzione non urgente. Parte da `develop`, torna in `develop`. |
| `hotfix/<descrizione>` | Correzione urgente di una release. Parte da `main`, torna in `main` e in `develop`. |

Nessun commit diretto su `main` o `develop`: solo merge da branch di lavoro.
