# CortexFlow Blog

La repository contiene i post che vengono caricati sul blog. Contiene:

- Blog creato con ghost contenente i post che vengono pubblicati nel blog
- Workflow CI/CD per crosspostare su Dev.to

# Installazione

1. Clonare la repository

```bash
https://github.com/CortexFlow/cortexflow-blog.git
```

2. Installare Ghost CLI tramite il comando

```bash
npm install ghost-cli@latest -g
```

Requisiti minimi per l'installazione:

- Node.js
- Npm


3. All'interno della repository eseguire Ghost con il comando

```bash
ghost start
```

# Comandi utili 

1. Build per versione in production:

```bash
gssg --url http://localhost:2368
```
Builda la cartella static da caricare su cortexflow.org/blog