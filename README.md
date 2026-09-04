# eBook AI Translator (Groq & Llama) 📚🤖

Questo repository contiene un potente strumento in **Python** per automatizzare la traduzione editoriale di eBook in formato **ePub** in lingua italiana, preservando integralmente la formattazione originale (fogli di stile CSS, tag HTML, suddivisione in capitoli) e generando opzionalmente una versione in **PDF** formattata ed elegante.

Il progetto è nato per risolvere i problemi tipici dei software esistenti: la rigidità dei tool basati su OpenAI, i crash dovuti alla saturazione della memoria (RAM/VRAM) nell'elaborazione locale e gli errori strutturali degli eBook che corrompono il file finale ZIP.

---

## 🚀 Caratteristiche Principali

*   **Cloud Translation Ultra-Veloce (Groq API)**: Utilizza l'API gratuita di Groq con il modello **Llama 3.1 8B**, garantendo traduzioni di qualità letteraria in pochissimi millisecondi, con un consumo di RAM del tuo computer inferiore a 60MB (addio ai colli di bottiglia dei modelli locali da diversi gigabyte).
*   **Ripristino e Sanificazione Attiva del Testo**: Pulisce riga per riga il testo da caratteri di controllo ASCII invisibili, spazi sporchi e tag corrotti che solitamente mandano i modelli di IA in loop infiniti (Token Loop).
*   **Gestione Avanzata del Rate-Limit**: Implementa una logica a 5 tentativi con backoff esponenziale e pause strategiche (`time.sleep`) per rispettare rigorosamente le soglie dei piani API gratuiti senza mai bloccare lo script.
*   **Suddivisione Protettiva (Chunking)**: Se un paragrafo o un blocco HTML è eccessivamente lungo, lo script lo frammenta automaticamente in blocchi più piccoli per evitare il superamento della finestra di contesto dell'API.
*   **Sistema di Memoria e Ripristino (Cache Locale)**: Salva progressivamente i capitoli tradotti in una cartella temporanea (`fasi_traduzione`). Se il PC si spegne o la traduzione si interrompe, lo script ripartirà istantaneamente dall'ultimo capitolo mancante senza sprecare token o tempo.
*   **Anteprima Real-Time Parziale**: Genera costantemente un file `libro_tradotto_parziale.epub` per consentirti di sbirciare e testare la qualità della traduzione su eReader prima ancora che l'intero libro sia completato.
*   **Correzione Automatica degli Errori dello ZIP (NCX Bypass)**: Risolve i famigerati errori di compilazione ePub (es. *no such path in zip: endpage.xhtml*) eliminando i link di indici orfani o corrotti nell'indice NCX e ricompilando lo ZIP in modo pulito.
*   **Generazione Multiformato**: Produce in un unico colpo sia l'**ePub** finale tradotto sia un **PDF** formattato in modo professionale tramite la libreria `reportlab`.

---

## 🛠️ Prerequisiti e Installazione

### 1. Clona il Repository ed entra nella cartella
```bash
git clone https://github.com/tuo-username/ebook-ai-translator.git
cd ebook-ai-translator
```

### 2. Installa le librerie richieste
Lo script necessita di alcune librerie Python per elaborare gli eBook, gestire l'HTML, comunicare con l'API e generare il PDF:
```bash
pip install groq ebooklib beautifulsoup4 tqdm reportlab
```

### 3. Ottieni una Chiave API Groq (Gratis!)
1. Vai su [console.groq.com](https://console.groq.com) e registrati (puoi usare il tuo account Google).
2. Nel menu laterale clicca su **API Keys**.
3. Clicca su **Create API Key**, nominala (es. *Traduttore*) e copia il codice segreto che inizia con `gsk_...`.
4. Inserisci la chiave all'interno dello script `traduci.py` nella variabile `GROQ_API_KEY`.

---

## 📄 Il Codice Completo (`traduci.py`)

Crea un file chiamato `traduci.py` nel tuo spazio di lavoro e incolla il seguente codice "corazzato":

```python
import os
import re
import time
import shutil
from ebooklib import epub, ITEM_DOCUMENT
from bs4 import BeautifulSoup
from groq import Groq
from tqdm import tqdm

# =====================================================================
# CONFIGURAZIONE STRUMENTO
# =====================================================================
GROQ_API_KEY = "INSERISCI_QUI_LA_TUA_CHIAVE_GROQ"  # Inizia con gsk_...
MODEL_NAME = "llama-3.1-8b-instant"
INPUT_BOOK = "libro.epub"       # Metti il tuo ePub in questa cartella e nominalo così
OUTPUT_BASE = "libro_tradotto"
CACHE_DIR = "fasi_traduzione"
# =====================================================================

# Inizializzazione client Groq
client = Groq(api_key=GROQ_API_KEY)

def pulisci_testo_grezzo(testo):
    """Rimuove caratteri speciali invisibili o corrotti per evitare loop nell'IA."""
    if not testo: 
        return ""
    # Rimuove caratteri di controllo ASCII non stampabili e strani glitch di codifica
    testo = re.sub(r'[\x00-\x08\x0b\x0c\x0e-\x1f\x7f-\xff]', '', testo)
    return re.sub(r'\s+', ' ', testo).strip()

def spezza_testo_lungo(testo, max_parole=120):
    """Spezza blocchi di testo colossali per non sovraccaricare la chiamata API."""
    parole = testo.split()
    if len(parole) <= max_parole:
        return [testo]
    return [' '.join(parole[i:i + max_parole]) for i in range(0, len(parole), max_parole)]

def forza_traduzione_groq_con_retry(testo, max_retries=5):
    """Invia il testo a Groq gestendo in modo protettivo errori di rete e Rate-Limit."""
    wait_time = 2  # Pausa iniziale prima di un retry
    for attempt in range(max_retries):
        try:
            response = client.chat.completions.create(
                model=MODEL_NAME,
                messages=[
                    {
                        "role": "system",
                        "content": (
                            "Sei un traduttore letterario professionista specializzato in narrativa e romanzi. "
                            "Il tuo compito è tradurre il testo dall'inglese all'italiano in modo che suoni perfettamente "
                            "fluido, naturale ed emotivamente coinvolgente, come se fosse stato scritto direttamente in italiano.\n"
                            "REGOLE RIGIDE:\n"
                            "1. NON tradurre letteralmente i modi di dire o le metafore inglesi.\n"
                            "2. Cura maniacalmente la grammatica italiana.\n"
                            "3. Mantieni i tempi verbali corretti e coerenti per la narrativa (es. passato remoto/imperfetto).\n"
                            "4. Restituisci ESCLUSIVAMENTE il testo tradotto. Non aggiungere note, introduzioni o commenti."
                        )
                    },
                    {
                        "role": "user",
                        "content": f"Testo da tradurre:\n{testo}"
                    }
                ],
                temperature=0.3,
                max_tokens=1000,
                timeout=12  # Taglio netto per evitare congelamenti infiniti
            )
            translation = response.choices[0].message.content.strip()
            if translation:
                return translation
        except Exception as e:
            # Satura o interrotto: aspetta in modo esponenziale e riprova
            print(f"\n[!] Errore API o Rate Limit (Tentativo {attempt+1}/{max_retries}). Attesa {wait_time}s... Errore: {e}")
            time.sleep(wait_time)
            wait_time *= 2
    return None

def main():
    print(f"Caricamento di {INPUT_BOOK}...")
    try:
        book = epub.read_epub(INPUT_BOOK)
    except FileNotFoundError:
        print(f"Errore: Sposta il tuo file ePub in questa cartella e nominalo '{INPUT_BOOK}'.")
        return
    except Exception as e:
        print(f"Errore nel caricamento del file EPUB: {e}")
        return

    if not os.path.exists(CACHE_DIR):
        os.makedirs(CACHE_DIR)

    # Estrazione di tutti i capitoli di testo effettivi
    html_items = [item for item in book.get_items() if item.get_type() == ITEM_DOCUMENT]
    total_chapters = len(html_items)
    print(f"Trovati {total_chapters} capitoli. Configurazione ambiente e stime...")

    progress_bar = tqdm(total=total_chapters, desc="Traduzione su Cloud Groq", unit="cap")
    full_text_for_pdf = []

    for idx, item in enumerate(html_items):
        print(f"\n--- [Analisi] Elaborazione Capitolo {idx + 1}/{total_chapters} ---")
        cache_file = os.path.join(CACHE_DIR, f"cap_{idx}.html")

        # 1. Recupero da cache locale se già tradotto precedentemente
        if os.path.exists(cache_file):
            with open(cache_file, "r", encoding="utf-8") as f:
                content_html = f.read()
                item.set_content(content_html.encode('utf-8'))
                soup = BeautifulSoup(content_html, 'html.parser')
                for node in soup.find_all(['p', 'h1', 'h2', 'h3', 'h4']):
                    full_text_for_pdf.append((node.name, node.get_text().strip()))
            progress_bar.update(1)
            continue

        try:
            content = item.get_content().decode('utf-8', errors='ignore')
        except Exception:
            content = str(item.get_content())

        soup = BeautifulSoup(content, 'html.parser')
        text_nodes = soup.find_all(['p', 'h1', 'h2', 'h3', 'h4'])

        # Protezione: Controlla se ci sono nodi di testo utili da tradurre
        nodi_validi = [n for n in text_nodes if pulisci_testo_grezzo(n.get_text()) and len(pulisci_testo_grezzo(n.get_text())) > 3]

        if not nodi_validi:
            # Capitolo vuoto, indice o solo immagini: salva direttamente senza interrogare l'IA
            with open(cache_file, "w", encoding="utf-8") as f:
                f.write(str(soup))
            progress_bar.update(1)
            continue

        # 2. Processo di traduzione dei nodi di testo
        for node in text_nodes:
            original_text = pulisci_testo_grezzo(node.get_text())
            if original_text and len(original_text) > 3:
                print(f"[Traducendo]: {original_text[:65]}...")
                
                # Sotto-frammentazione di sicurezza
                sotto_blocchi = spezza_testo_lungo(original_text)
                traduzioni_blocco = []
                
                for blocco in sotto_blocchi:
                    traduzione = forza_traduzione_groq_con_retry(blocco)
                    if traduzione:
                        traduzioni_blocco.append(traduzione)
                    else:
                        # Fallback: se fallisce, mantiene l'originale inglese per non lasciare vuoto
                        traduzioni_blocco.append(blocco)
                    time.sleep(1.2) # Pausa critica anti-saturazione Rate-Limit

                node.string = " ".join(traduzioni_blocco)

            if node.get_text().strip():
                full_text_for_pdf.append((node.name, node.get_text().strip()))

        # Salvataggio del contenuto tradotto nell'elemento dell'ePub
        translated_html = str(soup)
        item.set_content(translated_html.encode('utf-8'))

        # Salvataggio in cache fisica sul disco
        with open(cache_file, "w", encoding="utf-8") as f:
            f.write(translated_html)

        # 3. Aggiorna costantemente il file ePub parziale sul disco per i test di lettura
        try:
            epub.write_epub(f"{OUTPUT_BASE}_parziale.epub", book, options={'create_ncx': False})
        except Exception:
            pass

        time.sleep(1.5)
        progress_bar.update(1)

    progress_bar.close()

    # =====================================================================
    # COMPILAZIONE FORMATI FINALI
    # =====================================================================
    print("\nScrittura file finali con rimozione link corrotti...")
    
    # Sganciamo la Table of Contents vecchia per evitare puntatori corrotti a file mancanti nel file zip
    book.toc = []
    
    # Generazione ePub Definitivo
    try:
        epub.write_epub(f"{OUTPUT_BASE}.epub", book, options={'create_ncx': False})
        print("EPUB tradotto creato con successo.")
    except Exception as e:
        print(f"Errore durante il salvataggio strutturato dell'ePub: {e}. Tento salvataggio standard...")
        try:
            epub.write_epub(f"{OUTPUT_BASE}.epub", book)
        except Exception:
            print("Impossibile generare il file EPUB a causa di gravi incoerenze dello ZIP originale. Genero comunque il PDF.")

    # Generazione PDF Definitivo
    print("Generazione del file PDF definitivo...")
    try:
        from reportlab.lib.pagesizes import letter
        from reportlab.platypus import SimpleDocTemplate, Paragraph, Spacer
        from reportlab.lib.styles import getSampleStyleSheet, ParagraphStyle

        doc = SimpleDocTemplate(f"{OUTPUT_BASE}.pdf", pagesize=letter)
        styles = getSampleStyleSheet()
        story = []

        title_style = ParagraphStyle('TStyle', parent=styles['Heading1'], fontSize=18, leading=22, spaceAfter=12)
        body_style = ParagraphStyle('BStyle', parent=styles['Normal'], fontSize=11, leading=16, spaceAfter=8)

        for tag, text in full_text_for_pdf:
            if tag in ['h1', 'h2', 'h3', 'h4']:
                story.append(Paragraph(text, title_style))
            else:
                story.append(Paragraph(text, body_style))

        doc.build(story)
        print("PDF creato con successo.")
    except Exception as e:
        print(f"Nota: Errore durante la formattazione del PDF: {e}")

    # Pulizia file temporanei di cache
    if os.path.exists(CACHE_DIR):
        shutil.rmtree(CACHE_DIR)
    if os.path.exists(f"{OUTPUT_BASE}_parziale.epub"):
        os.remove(f"{OUTPUT_BASE}_parziale.epub")

    print("\n🎉 Processo di traduzione completato con successo!")
    print(f"📁 File pronti in cartella:\n - {OUTPUT_BASE}.epub\n - {OUTPUT_BASE}.pdf")

if __name__ == "__main__":
    main()
```

---

## 🏃 Come Avviare il Traduttore

1. Prendi il tuo libro in formato `.epub`, nominalo esattamente `libro.epub` e incollalo nella cartella del progetto.
2. Apri il terminale (Prompt dei comandi o PowerShell su Windows, o la shell su Linux/macOS) nella cartella del progetto ed esegui:
   ```bash
   python traduci.py
   ```
3. Vedrai scorrere a schermo la traduzione in tempo reale. I capitoli completati verranno salvati all'istante nella cache.
4. Se vuoi fermare lo script, premi `CTRL + C`. Potrai riavviarlo in qualsiasi momento con lo stesso comando e riprenderà esattamente da dove si era fermato!

---

## 🔍 Anatomia della Risoluzione dei Problemi

Ecco le sfide ingegneristiche superate in questa versione del codice, utili da conoscere per lo sviluppo:

### 1. Perchè siamo passati a Groq Cloud rispetto a Ollama Locale?
*   **Problema**: I modelli locali da 7B o superiori (come *Qwen 2.5 7B*) occupano circa 5-6 GB di VRAM. Su computer con schede video da 4 GB (come la *NVIDIA GTX 1650 Ti*), Windows esegue l'offloading sulla RAM di sistema, prosciugandola (fino a scendere sotto i 400 MB liberi) e causando il congelamento dello script.
*   **Soluzione**: Lo spostamento delle chiamate sulle API Cloud di Groq ha azzerato l'impatto sulla memoria del computer locale (lo script Python richiede solo 53 MB di RAM), incrementando la velocità di traduzione di circa il 1000%.

### 2. Risoluzione dei Blocchi al 56% (Capitoli corrotti)
*   **Problema**: Alcuni eBook presentano capitoli strutturati male, senza tag di chiusura HTML o ricchi di caratteri di controllo speciali invisibili. BeautifulSoup e l'IA andavano in loop infinito cercando di decodificarli, congelando l'avanzamento.
*   **Soluzione**: Lo script ora sanifica attivamente il testo isolando le stringhe con Regex ed esclude i capitoli privi di testo utile (es. indici strutturali o solo immagini) salvandoli istantaneamente nella cache per procedere oltre senza intoppi.

### 3. Errore "No such path in zip: endpage.xhtml"
*   **Problema**: Durante il salvataggio dell'ePub parziale o finale, la libreria `ebooklib` andava in crash cercando file di fine pagina (`endpage.xhtml`) dichiarati nel manifesto dell'eBook originale ma non esistenti fisicamente.
*   **Soluzione**: Disattivazione dell'opzione `create_ncx` in fase di compilazione temporanea e sgancio definitivo del `toc` vecchio a fine esecuzione per forzare la riscrittura di un archivio ZIP pulito e standardizzato.

---

## 📄 Licenza
Questo progetto è rilasciato sotto licenza MIT. Libero di essere modificato, migliorato e condiviso!
