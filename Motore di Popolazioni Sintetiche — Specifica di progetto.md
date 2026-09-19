# Motore di Popolazioni Sintetiche

## Specifica di progetto

2026-09-19 · @Luca Bianchi

## 1. Sintesi

Costruiamo un **motore che genera e fa vivere popolazioni di utenti sintetici**, e li mette a parlare con un qualunque sistema conversazionale.

L'artefatto centrale non è un test, né un chatbot, né un agente: è **la popolazione**. Un oggetto che si genera una volta, si versiona, si riusa, e si punta su bersagli diversi. Il motore che la fa vivere è il vero prodotto; tutto ciò che ci si costruisce sopra è un'applicazione.

Il motore serve due scenari già identificati, che usano lo stesso nucleo senza modifiche:

1. **Banco di prova.** Un chatbot (tipicamente RAG, in contesto aziendale) viene stressato da centinaia di utenti sintetici al posto di centinaia di persone reali. Si registrano gli esiti e si classificano i fallimenti.
2. **Ciclo di miglioramento.** Un agente viene esposto alla popolazione, riceve feedback, riscrive le proprie istruzioni, e viene rivalutato su una fetta di popolazione che non ha mai visto. Di iterazione in iterazione migliora, e il miglioramento è verificabile.

La domanda centrale e non banale del progetto è una sola: **quanto sono credibili gli utenti finti?** Un utente sintetico generato da un LLM è educato, coerente e ben scritto; un utente vero è ambiguo, sgrammaticato, impaziente e fuori scope. Se la popolazione non è fedele, un bersaglio che la supera fallisce comunque in produzione, e lo strumento avrà fatto danno invece che bene.

A questa domanda il progetto risponde con dati reali. Esistono corpora pubblici di conversazioni tra utenti veri e chatbot, e da lì si ricava sia il materiale per costruire la popolazione sia il metro per giudicarla. La metrica di fedeltà è un test di Turing sulla popolazione: se un giudice non riesce a distinguere i turni sintetici da quelli reali, la popolazione è buona.

Tutto gira su inferenza di modelli già esistenti. **Nessun addestramento di pesi.** L'esecuzione di default è locale su GPU da 16 GB, a costo zero, e la sostituzione con un provider esterno è un cambio di configurazione.

## 2. Il punto di partenza

Questa sezione esiste perché tra sei mesi sarà utile ricordare perché il progetto è nato così.

**Profilo.** Data Engineer in consulenza da otto mesi, prima esperienza lavorativa. Esperienze precedenti sul lato AI Engineering, tra cui un'applicazione RAG. Intenzione dichiarata di restare Data Engineer almeno per un altro anno, e di spostarsi verso l'AI Engineering in seguito, considerando le due cose collegate.

**Obiettivo dichiarato, in ordine di priorità.**

1. Un progetto personale che sia **interessante per me**, prima ancora che per un futuro datore di lavoro. Questo è il criterio dominante e non è negoziabile.
2. Qualcosa che si possa iniziare come soluzione embrionale e arricchire per passi successivi.
3. Qualcosa che abbia la forma di una **base**, un substrato su cui costruire *n* cose sopra. L'analogia usata fin dall'inizio è il game engine, e più avanti l'espressione "un mondo a sé".
4. Secondariamente, competenze spendibili in offerte di lavoro da AI Engineer in Italia.

**Vincoli tecnici.**

- GPU da 16 GB (RTX 5070 Ti). Vincola a modelli densi fino a circa 14B in quantizzazione a 4 bit, o MoE equivalenti.
- **Nessun addestramento di pesi.** Escluso esplicitamente per mancanza di risorse hardware. Si usano modelli esistenti.
- Costo zero in sviluppo, con sostituibilità futura verso provider esterni senza riscritture.

**Non richiesto e scartato lungo il percorso:** un assistente personale generalista in stile JARVIS, e un simulatore sociale generativo.

## 3. Come ci siamo arrivati

Il percorso è servito a restringere, e vale la pena registrarlo perché ogni scarto contiene un'informazione.

**Primo giro: cinque famiglie di "motore".** Sono state mappate cinque forme diverse che può assumere una base in ambito AI: motori di esecuzione (runtime durevoli, sistemi operativi per agenti, daemon guidati da eventi), motori di linguaggio (LLM usato come compilatore, DSL per programmi LLM, motori di contratti), motori di mondo (interactive fiction, simulazione sociale, master di gioco), motori di interfaccia (UI generativa, canvas a grafo), motori di capacità e verifica (gateway di tool, tool auto-estendenti, debugger a viaggio nel tempo, popolazioni di utenti simulati).

**Scarti immediati** perché percepiti come fini a sé stessi: daemon ambientale, motore di contratti, master di gioco, canvas a grafo, gateway di tool, debugger a viaggio nel tempo.

**Scarti su analisi.** Un sistema operativo per agenti è stato escluso perché contiene un problema sostanzialmente irrisolto (l'annullamento di azioni con effetti reali) e quindi non ha un confine di scope. L'LLM come compilatore è stato messo da parte perché il suo valore dipende interamente dalla qualità del modello, il che lo renderebbe un progetto obbligato su API frontier, in contraddizione con il vincolo di costo zero. Il DSL con ottimizzatore di prompt è stato rimandato perché **contiene la valutazione come prerequisito**: non si può ottimizzare ciò che non si sa misurare. Da qui l'osservazione che ha orientato tutto il resto: quando un progetto ne contiene un altro come prerequisito, il prerequisito è il progetto giusto.

**La verifica su JARVIS.** L'idea dell'assistente onnipotente è stata esaminata e archiviata con tre motivazioni. Primo, l'asimmetria della verifica: un assistente corretto al 95% è inutile per qualunque cosa abbia conseguenze, perché controllare costa quanto fare. Secondo, il valore è proporzionale alla superficie di integrazione, e quella superficie è quasi tutta idraulica noiosa. Terzo, ciò che rende JARVIS interessante (memoria personale e iniziativa) era già stato scartato in questa conversazione, il che suggerisce che l'attrazione fosse estetica più che funzionale.

**Una deviazione utile.** Il progetto "utenti simulati" è stato prima proposto come strumento di valutazione e red teaming, e la reazione è stata di rigetto: era diventato un processo aziendale di QA, non un mondo. È stato riformulato mettendo **la popolazione al centro come artefatto** e gli usi come applicazioni. Questa inversione è la forma definitiva del progetto ed è quella corretta.

**Il buco residuo e la sua chiusura.** Restava aperta la fragilità principale: utenti finti fatti da un LLM non assomigliano a utenti veri, e senza un metro esterno il progetto resta una demo. La ricerca di corpora pubblici di conversazioni reali ha chiuso questo buco e ha fornito al progetto la sua metrica fondante.

## 4. Definizione del progetto

### Cosa costruiamo

Un sistema composto da tre strati concentrici.

**Lo strato centrale: la popolazione.** Un insieme di *persone* sintetiche, ciascuna definita da una scheda strutturata (obiettivo, conoscenza posseduta, stile, intenzione, tratti comportamentali). Le persone sono dati, non codice. Una popolazione ha un nome, un seme, una versione, e si conserva.

**Lo strato intermedio: il motore.** Ciò che genera le popolazioni, le fa parlare con un bersaglio, registra cosa è successo, giudica gli esiti e analizza i fallimenti. È qui che sta il valore riutilizzabile.

**Lo strato esterno: le applicazioni.** Configurazioni del motore che rispondono a domande diverse. Il banco di prova e il ciclo di miglioramento sono le prime due; altre arriveranno.

### Perché è una base, nel senso letterale

La prova che si tratta di una base e non di un'applicazione è che dallo stesso nucleo escono cose che sembrano progetti diversi: un banco di prova per un chatbot aziendale, un ciclo di auto-miglioramento di un agente, un confronto tra due architetture di agente a parità di compito, un generatore di report di robustezza, un benchmark di modelli locali sul dialogo multi-turno. Nessuna di queste richiede di toccare il motore.

### Il bersaglio

Il **bersaglio** è il sistema sotto esame. Il motore ne sa una cosa sola: gli si mandano messaggi, risponde, talvolta chiama dei tool. Grazie a questa interfaccia minima il bersaglio può essere un modello locale, un'API, un agente open source, un server MCP, o un sistema costruito in azienda.

## 5. Principi di progettazione

Sette regole, da rileggere ogni volta che una decisione sembra difficile.

1. **Niente astrazione prima di due casi reali.** Il generico si scopre, non si progetta. Un'interfaccia universale disegnata prima di avere due usi funzionanti è architettura a vuoto, ed è il modo più rapido di uccidere questo progetto.
2. **La popolazione è dati, il motore è codice.** Aggiungere un comportamento nuovo deve significare scrivere una scheda, non un ramo condizionale.
3. **Ogni numero deve avere un metro esterno.** Una metrica prodotta da un LLM e mai confrontata con un giudizio umano è rumore travestito da misura.
4. **Separazione costruzione/validazione, sempre.** Una fetta di popolazione e una fetta di dati reali restano fuori da ogni ciclo di costruzione o ottimizzazione. Senza questo, ogni miglioramento osservato è autoinganno.
5. **Tutto ciò che accade viene persistito.** Ogni turno, ogni chiamata, ogni verdetto, con la versione di chi lo ha prodotto. Un esperimento non riproducibile è un aneddoto.
6. **Interfaccia OpenAI-compatible ovunque.** Ogni chiamata a un modello passa da un unico punto configurabile. La sostituzione di un provider è una riga di configurazione.
7. **Nessuna persona senza ipotesi.** Non si aggiunge un tipo di utente se non si sa quale domanda sta ponendo al bersaglio. Altrimenti si costruisce uno zoo.

## 6. Gli scenari d'uso

### Scenario A — Il banco di prova (primo da realizzare)

**Domanda:** questo chatbot regge l'uso di utenti veri?

**Perché è il primo:** è concreto, il valore è comprensibile in dieci secondi ("duecento utenti finti al posto di duecento persone"), e corrisponde a un bisogno reale che si incontra in consulenza. Inoltre permette di giudicare a occhio se le risposte sono buone, cosa che rende possibile validare il giudice.

**Flusso:** si sceglie un bersaglio, si genera o si carica una popolazione, si esegue, si ottiene un rapporto con tasso di successo per tipo di persona, categorie di fallimento e conversazioni esemplari da leggere.

### Scenario B — Il ciclo di miglioramento (secondo)

**Domanda:** un agente può migliorare da solo se esposto ripetutamente a una popolazione?

**Cosa significa "migliorare":** non pesi. L'agente cambia le proprie istruzioni, i propri esempi, la propria strategia. È testo che evolve, il che è compatibile con il vincolo hardware e non richiede alcun addestramento.

**Il ciclo:** si esegue la popolazione di costruzione contro l'agente, si raccolgono i fallimenti etichettati, si fa riscrivere all'agente le proprie istruzioni sulla base di quei fallimenti, si riesegue.

**Il controllo indispensabile:** la valutazione avviene anche sulla **popolazione di validazione**, mai usata per l'ottimizzazione. Se il successo sale su entrambe, il miglioramento è reale. Se sale solo sulla popolazione di costruzione, l'agente ha imparato a compiacere il simulatore, e il risultato va buttato. Senza questo controllo lo scenario B non significa nulla.

### Scenari futuri (non da progettare ora)

Confronto tra architetture di agente a parità di compito. Gate di regressione prima di modificare un prompt. Rapporti di robustezza. Benchmark di modelli locali sul dialogo multi-turno con tool.

## 7. Architettura

Nove componenti. I primi sei sono il v0; gli ultimi tre arrivano dopo.

**1. Generatore di popolazione.** Da uno schema e una distribuzione produce N schede persona. Le distribuzioni (lunghezza dei turni, tasso di ambiguità iniziale, propensione all'abbandono, mix linguistico) sono parametri, e nella versione matura vengono stimati dai dati reali anziché inventati.

**2. Attore.** Fa vivere una singola persona, turno per turno. Il requisito non banale: deve **reagire davvero** a ciò che il bersaglio risponde, non recitare un copione. Se il bersaglio fraintende, la persona deve riformulare come riformulerebbe quel tipo di utente; se il bersaglio è inutile tre volte, la persona che ha poca pazienza deve abbandonare.

**3. Adattatore del bersaglio.** L'unico punto in cui il motore tocca il sistema sotto esame. È l'astrazione che rende il progetto una base.

**4. Orchestratore.** Esegue molte conversazioni concorrenti, gestisce timeout, tentativi, limiti di concorrenza, e scrive le tracce. È il componente che decide se una sessione dura minuti o ore.

**5. Registro delle tracce.** Persistenza di tutto quello che è successo, con le versioni di tutto ciò che lo ha prodotto.

**6. Giudice.** Etichetta ogni conversazione conclusa secondo una tassonomia esplicita. Ha una versione, e la versione entra nel registro.

**7. Validatore di fedeltà.** Il test di Turing sulla popolazione. Componente separato, perché risponde a una domanda sul motore stesso e non sul bersaglio.

**8. Analizzatore.** Raggruppa i fallimenti in cluster, così che cento conversazioni rotte diventino sei problemi. Calcola il diff tra due sessioni.

**9. Ottimizzatore.** Prende i fallimenti etichettati e riscrive le istruzioni del bersaglio. Esiste solo nello scenario B, e solo dopo che il giudice è stato validato.

### Flusso

```
schema → [Generatore] → popolazione (build | holdout)
                              ↓
         [Attore] ←───────────┘
             ↕  turni
     [Adattatore] ↔ BERSAGLIO
             ↓
     [Orchestratore] → [Registro tracce]
                              ↓
                         [Giudice] → verdetti
                              ↓
                       [Analizzatore] → cluster, diff, rapporto
                              ↓
                       [Ottimizzatore] → nuove istruzioni del bersaglio
                              └──────── (rientra come nuova versione)

[Validatore di fedeltà]: popolazione + corpus reale → tasso di smascheramento
```

## 8. Interfacce e contratti

Tre contratti. Vanno congelati presto, perché sono ciò che rende il motore riusabile.

### L'adattatore del bersaglio

Minimo indispensabile, e niente di più:

```python
class TargetAdapter(Protocol):
    async def start(self, conversation_id: str) -> None: ...
    async def send(self, conversation_id: str, message: str) -> TargetReply: ...
    async def end(self, conversation_id: str) -> None: ...

    @property
    def version(self) -> str: ...   # entra nel registro di ogni sessione
```

`TargetReply` contiene il testo della risposta, le eventuali chiamate a tool, la latenza e i metadati grezzi. Il bersaglio può essere stateful (gestisce lui la conversazione) o stateless (l'adattatore gli rimanda la storia): la differenza resta dentro l'adattatore e il motore non la vede.

### La scheda persona

```python
class Persona(BaseModel):
    id: str
    population_id: str

    obiettivo: str                  # cosa vuole ottenere
    criterio_successo: str          # come sapremmo che l'ha ottenuto
    conoscenza: dict                # cosa sa già (e cosa crede erroneamente)

    registro: str                   # formale, colloquiale, brusco
    competenza_dominio: int         # 1-5
    competenza_tecnologica: int     # 1-5
    pazienza: int                   # turni prima di abbandonare
    precisione_iniziale: int        # quanto è specifica la prima richiesta
    lingua: str                     # it, en, mista

    intenzione: Literal["cooperativa", "confusa", "incoerente", "avversariale"]
    tratti: list[str]               # cambia idea, insiste, va fuori tema

    split: Literal["build", "holdout"]
```

Il campo `criterio_successo` è il più importante e il più facile da trascurare: senza di esso non esiste un esito, e senza esito non esiste una metrica.

### Il formato della traccia

JSONL, una riga per conversazione, con: identificativi di sessione, persona e bersaglio; la sequenza completa dei turni con ruolo, contenuto, chiamate a tool, latenza e token; l'esito; e un blocco di provenienza con i modelli usati, le loro versioni, i parametri di campionamento e il commit del codice.

## 9. Modello dei dati

SQLite, un file unico. Lo schema è deliberatamente relazionale perché le domande che si vorranno fare sono relazionali ("il tasso di successo per intenzione, confrontato tra due versioni del bersaglio").

```sql
populations(id, name, seed, schema_version, params_json, created_at)
personas(id, population_id, spec_json, split)

targets(id, name, kind, config_json, version)

runs(id, population_id, target_id, started_at, finished_at,
     config_json, code_commit)

conversations(id, run_id, persona_id, status, turn_count, outcome)
turns(id, conversation_id, ord, role, content, tool_calls_json,
      latency_ms, tokens_in, tokens_out)

judgments(conversation_id, judge_version, labels_json, rationale)
human_labels(conversation_id, labeler, labels_json, labeled_at)

fidelity_trials(id, population_id, corpus, sample_json,
                true_origin, guess, guesser)
```

Tre osservazioni sullo schema.

`split` sta sulla persona e non sulla sessione, perché l'appartenenza alla fetta di validazione è una proprietà permanente della persona: se cambiasse tra un esperimento e l'altro, la separazione non varrebbe più.

`judge_version` e `code_commit` esistono perché prima o poi si guarderà un risultato di tre mesi prima chiedendosi con cosa era stato prodotto.

`human_labels` è una tabella separata da `judgments` e non una colonna, perché le due cose vanno confrontate, non fuse.

## 10. Le fonti di verità

Sono ciò che distingue questo progetto da una demo. Servono a due scopi distinti: come **materiale** per costruire la popolazione, e come **metro** per giudicarla.

### Corpora di conversazioni reali con chatbot

**WildChat** (Allen AI) è la fonte primaria. Circa un milione di conversazioni tra utenti reali e ChatGPT, raccolte offrendo accesso gratuito in cambio di consenso esplicito, rilasciate sotto licenza ODC-BY. È multilingue e contiene metadati geografici, quindi la fetta italiana è isolabile. La qualità che serve a noi è che i curatori documentano esplicitamente la presenza di richieste ambigue, alternanza di codice linguistico e cambi di argomento: esattamente i comportamenti che un utente sintetico non ha spontaneamente.

**LMSYS-Chat-1M** come fonte secondaria. Circa un milione di conversazioni reali con venticinque modelli, raccolte da oltre duecentomila indirizzi IP distinti, con ampia copertura linguistica. Richiede di accettare un accordo di licenza prima dell'accesso, da leggere se un giorno si pensa di pubblicare risultati.

### Corpora di supporto clienti e task-oriented

Più vicini allo scenario A. **ABCD** raccoglie oltre diecimila conversazioni tra clienti e operatori su problemi reali. **MSDialog** e **Ubuntu Dialogue Corpus** contengono conversazioni di supporto tecnico prese da forum veri, con tutta la sgrammaticatura e l'impazienza del caso.

**MultiWOZ** e **Taskmaster** sono grandi e ben annotati ma vanno usati con cautela per il nostro scopo: sono raccolti con il metodo Wizard-of-Oz, dove l'utente umano è guidato da una descrizione del compito, il che lascia pochissimo spazio a comportamenti fuori copione. Producono utenti diligenti. Utili per la struttura dei dialoghi, inadatti come metro di realismo.

### L'avvertimento sull'italiano

I dataset conversazionali nativamente italiani disponibili pubblicamente sono in larghissima parte **generati o scritti apposta** per il fine-tuning, non catturati da utenti veri. Costruire un generatore di utenti finti su dati già finti è un ragionamento circolare: si impara a imitare come un LLM immagina un utente. Per l'italiano, le fette italiane di WildChat e LMSYS sono praticamente l'unica fonte reale disponibile. Meno abbondanti dell'inglese, ma vere.

### Cosa si trasferisce e cosa no

Da questi corpora si prendono i **comportamenti** (come si apre una conversazione, come si riformula dopo un fraintendimento, quando si abbandona, come si mischiano le lingue), non i **contenuti**. I contenuti vengono dal dominio del bersaglio.

Resta consapevolmente aperta una domanda interessante: il modo in cui le persone si comportano con un assistente generalista si trasferisce davvero a un chatbot aziendale di dominio? Non c'è una risposta consolidata. Questo strumento permette almeno di porre la domanda, ed è un possibile approdo del progetto.

## 11. Il sistema di misura

Tre livelli di misura, che rispondono a tre domande diverse. Vanno costruiti in quest'ordine, perché ciascuno rende sensato il successivo.

### Livello 1 — La fedeltà della popolazione

**Domanda:** i miei utenti finti assomigliano a utenti veri?

**Metodo.** Si estraggono N turni da un corpus reale e N turni prodotti dalla popolazione, si mescolano, e si chiede a un giudice (prima umano, poi automatico) di indicare quali sono reali.

**La metrica è il tasso di smascheramento.** Il 50% significa indistinguibile; il 100% significa che la popolazione è palesemente finta. Un valore sotto il 70% è un buon traguardo di partenza.

**Regola di igiene:** metà del corpus reale si usa per costruire, metà resta chiusa per validare. Se si usa tutto per costruire, il test smette di misurare.

### Livello 2 — L'affidabilità del giudice

**Domanda:** posso fidarmi delle etichette che produce il sistema?

**Metodo.** Si etichettano **a mano circa cento conversazioni** e si misura l'accordo tra quelle etichette e quelle del giudice automatico, con una statistica di concordanza (kappa di Cohen o equivalente).

Questa è la tappa più noiosa dell'intero progetto ed è anche quella che lo rende reale. Se si salta, tutte le metriche successive sono numeri senza significato. È anche il lavoro che si può fare solo a mano: non c'è scorciatoia, ed è precisamente la competenza che manca alla maggior parte di chi costruisce sistemi LLM.

**Effetto collaterale prezioso:** etichettando cento conversazioni si scopre che la tassonomia di fallimento iniziale è sbagliata. Va bene così. La si riscrive e la si versiona.

### Livello 3 — Le metriche sul bersaglio

Solo dopo i primi due livelli.

- Tasso di successo del compito, complessivo e **per tipo di persona**
- Turni necessari al successo
- Tasso di abbandono
- Tasso di risposte non fondate
- Tasso di violazione per categoria, quando si introducono persone avversariali

Il valore non sta nel numero aggregato, che dice poco, ma nella **disaggregazione**: scoprire che il bersaglio funziona bene con utenti competenti e crolla con utenti che non sanno formulare la domanda è un risultato utile. Il numero aggregato lo nasconderebbe.

### Per lo scenario B

Si aggiunge una sola coppia di numeri, ma è quella decisiva: il tasso di successo sulla popolazione di costruzione e quello sulla popolazione di validazione, iterazione per iterazione. La divaricazione tra le due curve è la diagnosi di sovradattamento al simulatore.

## 12. Stack tecnico

**Linguaggio e ambiente.** Python 3.12, gestito con `uv`. Un solo progetto, niente Docker nel v0.

**Concorrenza.** `asyncio`. L'orchestratore è I/O-bound: decine di conversazioni concorrenti sono il caso normale.

**Schemi.** Pydantic per persone, risposte del bersaglio e verdetti. Gli schemi hanno una versione e la versione finisce nel registro.

**Persistenza.** SQLite per lo stato, JSONL per le tracce grezze. DuckDB in lettura sopra i JSONL quando l'analisi diventa pesante: è lo strumento che già si conosce e legge i file direttamente senza caricarli.

**Accesso ai modelli.** Client OpenAI-compatible, con l'endpoint in configurazione. Un unico modulo che incapsula ogni chiamata, con tentativi, timeout e conteggio dei token. È il punto che rende vero il principio 6.

**Servizio di inferenza locale.** Questa è la decisione infrastrutturale più importante del progetto, ed è facile sottovalutarla.

Il motore è un generatore di token su scala industriale: una sessione con duecento persone e dieci turni a testa produce migliaia di chiamate, e ogni turno ne richiede due (una per la persona, una per il bersaglio), più una per il giudizio finale. Con un servizio che evade le richieste in sequenza, una sessione dura ore; con un servizio che fa batching continuo e serve decine di richieste concorrenti sulla stessa scheda, dura minuti. La differenza decide se la sessione si lancia dieci volte al giorno o due volte alla settimana, e quindi se il progetto vive.

Tradotto: **vLLM (o equivalente con batching continuo) invece della configurazione predefinita di Ollama** appena si passa da una conversazione alla volta a molte in parallelo. Ollama va benissimo per il v0 a conversazione singola.

**Modelli.** Per persona, giudice e bersaglio servono ruoli distinti, possibilmente modelli distinti. Su 16 GB, un modello denso da circa 14B in quantizzazione a 4 bit occupa intorno ai 9 GB e lascia spazio al contesto. Nota importante sui ruoli: **il giudice non deve essere lo stesso modello del bersaglio**, per evitare che un sistema si giudichi da solo.

**Corpora.** Libreria `datasets` di HuggingFace, con una copia locale delle fette usate e la registrazione di quale fetta è finita in costruzione e quale in validazione.

**Cosa non si usa.** Nessun framework di agenti, nessuna libreria di valutazione preconfezionata. Il punto del progetto è che il motore si capisce riga per riga, e queste librerie nasconderebbero esattamente le decisioni che vale la pena prendere a mano.

## 13. Roadmap

| Versione | Cosa si costruisce | Il numero da guardare |
| --- | --- | --- |
| **v0** | Una persona scritta a mano, un bersaglio, loop multi-turno, traccia su file | Nessuno. Si leggono le conversazioni a mano: sono verosimili? |
| **v0.5** | 20 persone da schema, esecuzione concorrente, persistenza su SQLite, esito deterministico | Tasso di successo per tipo di persona |
| **v1** | Giudice LLM **e cento etichette fatte a mano** | **Accordo giudice-umano.** È la tappa che rende reale tutto il resto |
| **v1.5** | Validatore di fedeltà contro corpus reale | Tasso di smascheramento |
| **v2** | Generatore parametrizzato sui dati reali, persone avversariali, tassonomia versionata | Tasso di violazione per categoria; tasso di smascheramento che scende |
| **v2.5** | Analizzatore: cluster di fallimenti, diff tra sessioni | Numero di problemi distinti dietro N fallimenti |
| **v3** | Scenario B: ciclo di miglioramento con separazione build/holdout | Successo su build **e** su holdout, iterazione per iterazione |
| **v4** | Ottimizzatore automatico delle istruzioni del bersaglio | La divaricazione tra le due curve |

Il v0 è un weekend onesto. Il v1 si raggiunge in tre o quattro settimane di sere ed è il punto in cui il progetto smette di essere un giocattolo.

## 14. Rischi e trappole

**Generalizzare troppo presto.** Il rischio numero uno, perché è quello che sembra virtuoso. Antidoto: principio 1, applicato senza sconti.

**Lo zoo di persone.** Aggiungere il ventunesimo tipo di utente è gratificante e non dice niente di nuovo. Antidoto: principio 7.

**Fidarsi del giudice.** Saltare il v1 perché sembra burocrazia. È l'esatto contrario: senza quella tappa, tutto ciò che viene dopo è finzione numerica.

**Sovradattamento al simulatore.** Nello scenario B, l'agente impara a compiacere la popolazione invece di migliorare. Antidoto: la fetta di validazione, tenuta rigorosamente fuori.

**Il bersaglio troppo facile.** Se si rompe al primo colpo, il motore non discrimina e i risultati non informano. Serve un bersaglio che regga abbastanza da rendere i fallimenti interessanti.

**Il collo di bottiglia del servizio di inferenza.** Se una sessione dura ore, si smette di lanciarla. Da affrontare appena si passa alla concorrenza, non dopo.

**Deriva verso il QA.** Il rischio narrativo: il progetto può scivolare verso l'infrastruttura di controllo qualità e perdere ciò che lo rendeva interessante. Antidoto: tenere la popolazione al centro come artefatto, e coltivare le persone avversariali, che è la parte in cui si attacca invece di verificare.

## 15. Cosa questo progetto non è

Utile quanto la definizione positiva.

- **Non è un framework di agenti.** Non costruisce agenti, li mette alla prova.
- **Non è una piattaforma di osservabilità generica.** Non compete con gli strumenti di tracing esistenti. Registra ciò che gli serve e nulla più.
- **Non comporta addestramento.** Nessun peso viene modificato. Ciò che evolve è testo: istruzioni, esempi, schede persona.
- **Non è un prodotto.** Non ha utenti da servire, non ha uptime, non ha una roadmap commerciale.
- **Non è un assistente.** JARVIS è stato esaminato e archiviato, con motivazioni registrate alla sezione 3.

## 16. Prossimi passi e domande aperte

### Da fare per primo, prima di scrivere codice

1. **Scaricare WildChat e leggerne cento conversazioni a mano**, isolando la fetta italiana. Non per estrarre statistiche: per farsi un'idea viscerale di come parlano le persone vere. Questo passo condiziona tutto ciò che viene dopo e non è delegabile.
2. **Scegliere il primo bersaglio** (vedi domanda aperta sotto).
3. **Scrivere dieci schede persona a mano**, prima di automatizzare qualunque generazione. Servono a scoprire quali campi la scheda deve davvero avere.
4. Solo a questo punto, il v0.

### Domande aperte da risolvere

**Quale primo bersaglio.** È la decisione che determina la forma del v0, perché condiziona quali persone abbia senso generare e cosa conti come successo. Le opzioni sono un modello locale con tool, un piccolo sistema RAG costruito ad hoc su un dominio noto, o un agente open source già esistente. La seconda è probabilmente la più vicina allo scenario A e quella che permette di giudicare gli esiti con sicurezza.

**Lingua di lavoro.** Italiano o inglese. L'italiano è più aderente al caso d'uso aziendale ma ha meno dati reali disponibili; l'inglese ha corpora molto più ricchi. Possibile compromesso: motore agnostico, prima popolazione in italiano, validazione di fedeltà fatta prima in inglese dove i dati abbondano.

**Quanto il generatore debba essere consapevole del dominio.** Una persona che interroga un chatbot bancario ha bisogno di conoscenze bancarie plausibili. Quanta parte di questo va nella scheda e quanta in una descrizione del dominio separata e riusabile? Probabilmente separata, ma va verificato sul campo.

**Il nome.** Il progetto ha bisogno di un nome proprio, che non è un dettaglio: dà identità a un lavoro lungo. Candidati da valutare: *Coorte*, *Agorà*, *Folla*, *Demos*.
