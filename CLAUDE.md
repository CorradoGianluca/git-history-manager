# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Stato del progetto

Estensione VS Code che riproduce il flusso di lavoro Git di IntelliJ IDEA (albero dei branch, Log con grafo e filtri, dettaglio dei commit, operazioni dalle viste). Scritta da zero, distribuita come `.vsix` (niente Marketplace). Windows, macOS e Linux supportati fin dall'inizio.

Il repository non contiene ancora codice: non esistono `package.json`, script di build o test. Non inventare comandi; aggiungili qui quando S0 li crea.

Fonte delle decisioni: documento tecnico v0.4 (fuori dal repository), `estensione-vscode-git-documento-tecnico.docx`. Ogni specifica S0–S6 è un documento separato con il proprio piano di implementazione.

## Decisioni trasversali (D1–D9)

Valgono per tutte le specifiche. Una deroga va dichiarata nella specifica e riportata nel documento tecnico.

| ID | Decisione |
|---|---|
| D1 | Due metà senza memoria condivisa: **extension host** (Node.js) parla con git e detiene lo stato; **webview** (UI) visualizza e invia comandi. Nessuno stato duplicato nella webview. |
| D2 | Webview in React + TypeScript, build con Vite. Extension host compilato con esbuild. Icone Codicons. Colori solo dalle variabili CSS `--vscode-*`. |
| D3 | Git tramite CLI nativa (`child_process.spawn`). L'API dell'estensione git integrata serve solo per: percorso dell'eseguibile, scoperta dei repository, eventi di cambio stato. Scartati: solo API integrata, isomorphic-git, nodegit/libgit2. |
| D4 | Il Log è una `WebviewView` nel pannello inferiore (spostabile in sidebar/secondary sidebar); un comando lo apre anche come `WebviewPanel` in un tab dell'editor. |
| D5 | Un unico bundle frontend, indipendente dal contenitore e responsive. |
| D6 | Multi-repository fin dall'inizio: ogni oggetto di dominio porta l'identità del repository (es. `{repoId, hash}`, `{repoId, name, kind}`). |
| D7 | Controllo sincronizzato dei branch come IntelliJ: branch comuni al primo livello, avviso di divergenza, rollback proposto sui fallimenti parziali. La proposta automatica di attivazione è da confermare. |
| D8 | Convive con il Source Control integrato, che deve restare attivo (D3 ne dipende). |
| D9 | Versioni minime di git e VS Code = quelle installate all'inizio dello sviluppo; fissate in S0 (`engines.vscode` + controllo all'attivazione). |

## Struttura del codice

Un unico pacchetto, un unico `.vsix`:

- `src/extension/` — extension host.
- `src/webview/` — UI React.
- `src/shared/` — tipi del protocollo host ↔ webview e del dominio, importati da entrambe le metà. Un cambio di contratto deve rompere la compilazione, non il runtime.

Output di build ignorati: `out/` (esbuild), `dist/` (Vite).

## Architettura dell'extension host (S0)

- **RepositoryRegistry**: scopre i repository via API git integrata e crea un `RepoContext` per ciascuno.
- **GitRunner**: unico punto che avvia git. Supporta `AbortSignal`, stdin, variabili d'ambiente per singolo comando. Registra ogni comando (durata, exit code) nella **Console** (Output Channel).
- **RepoContext**: uno per repository. Serializza le scritture, consente letture concorrenti e annullabili, tiene in cache HEAD, ref, upstream (ahead/behind) e status.
- **Parser**: funzioni pure output git → oggetti di dominio immutabili, testabili senza git.
- **StateStore**: unica fonte di verità per stato e impostazioni (incluso il controllo sincronizzato); emette eventi.
- **Refresh**: eventi dell'API integrata e delle operazioni invalidano solo la cache interessata, con debounce.

### Regole del livello git

- Output separato da NUL (`-z`, `%x00` nei `--format`) e `core.quotepath=false`.
- Liste lunghe di file/commit via stdin (`--stdin`, `--pathspec-from-file=-`): limite della riga di comando su Windows.
- Pochi processi: cache e comandi aggregati (es. un unico `git log` paginato).
- Scritture serializzate per repository (evita `index.lock`); letture superate annullate.
- `GIT_TERMINAL_PROMPT=0` sulle letture.
- Messaggi git forzati in inglese tramite le variabili di locale, per classificare gli errori.

### Errori

Ogni fallimento diventa un `GitError` con repository, comando, exit code, stderr e categoria: conflitto, lock, autenticazione, rifiuto del remoto (non fast-forward / lease rifiutato / rifiuto lato server), ref inesistente, altro. Le operazioni multi-repo riportano sempre l'esito repository per repository.

### Protocollo host ↔ webview

Tre tipi di messaggio: query con risposta, comandi, eventi dall'host. L'host invia gli eventi a tutte le webview aperte. Lo stato di vista (selezione, filtri, dimensioni) è locale a ogni istanza e sopravvive al reload. CSP restrittiva, risorse caricate solo dall'estensione.

### Estendibilità

- S1 definisce un **registro delle azioni** da cui leggono menu contestuali e toolbar: le operazioni delle specifiche successive si aggiungono registrando azioni, senza toccare il codice del Log.
- S2 definisce le **operazioni** come oggetti con repository, precondizioni, fotografia dello stato, esecuzione e rollback, avviati da un esecutore comune multi-repo.
- Force push solo come `--force-with-lease`; mai `--force`, neppure come ripiego.

## Specifiche

| Spec | Contenuto | Dipende da |
|---|---|---|
| S0 | Fondamenta (sopra) | — |
| S1 | Log in sola lettura: albero branch, lista commit con grafo, filtri, dettaglio e diff, tab multipli, multi-repo, registro azioni | S0 |
| S2 | Operazioni su branch e commit, controllo sincronizzato, rollback | S0, S1 (S4) |
| S3 | Fetch, update project, push, autenticazione | S2 |
| S4 | Finestra di commit, changelist, commit per hunk, amend, stash/shelve | S0 |
| S5 | Rebase interattivo, azioni di modifica della history, conflitti | S2, S4 (S3) |
| S6 | History di file e selezione, annotate | S0, S1 |

Ordine: S0 → S1 → S2 → S3; S4 in parallelo dopo S0; poi S5 e S6.

## Test (strategia di S0)

- Parser: test unitari su output git registrati.
- GitRunner e RepoContext: integrazione su repository reali in cartelle temporanee, in CI su Windows, macOS e Linux.
- Componenti React: test con protocollo simulato.
- Smoke test in un'istanza reale di VS Code.

## Branch e line ending

- `main`: solo release; ogni merge ha un tag `vX.Y.Z`.
- `develop`: integrazione, base di ogni nuovo lavoro.
- `feature/<spec>-<descrizione>` (es. `feature/s0-git-runner`) e `fix/<descrizione>`: da `develop`, tornano in `develop`.
- `hotfix/<descrizione>`: da `main`, torna in `main` e in `develop`.
- Nessun commit diretto su `main` o `develop`: solo merge da branch di lavoro.
- Messaggi di commit in inglese.

`.gitattributes` impone LF ovunque (CRLF solo per `*.cmd`/`*.bat`).
