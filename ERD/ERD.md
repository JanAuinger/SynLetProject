# ERD – Synthetic Lethality Datenbank

Ziel: Zu einem mutierten humanen Gen X alle synthetisch letalen Partner finden, nach Evidenz und Essentialität bewerten und prüfen, ob die Beziehung über Orthologe auch in *Rattus norvegicus* bzw. *E. coli K-12* vorkommt.

```mermaid
erDiagram
    ORGANISM ||--o{ GENE : "hat"
    GENE ||--o{ GENE_SYNONYM : "hat"
    GENE ||--o| ESSENTIALITY : "hat"
    ESSENTIALITY }o--o{ PUBLICATION : "belegt durch"
    GENE ||--o{ INTERACTION : "Interaktor A"
    GENE ||--o{ INTERACTION : "Interaktor B"
    INTERACTION_TYPE ||--o{ INTERACTION : "klassifiziert"
    DETECTION_METHOD ||--o{ INTERACTION : "nachgewiesen mit"
    PUBLICATION ||--o{ INTERACTION : "beschreibt"
    INTERACTION ||--o{ INTERACTION_PHENOTYPE : "zeigt"
    PHENOTYPE ||--o{ INTERACTION_PHENOTYPE : "tritt auf in"
    GENE ||--o{ ORTHOLOGY : "humanes Gen"
    GENE ||--o{ ORTHOLOGY : "Ortholog"

    ORGANISM {
        int    taxid PK "NCBI Taxonomy ID, z.B. 9606, 10116, 316407"
        string name "z.B. Homo sapiens"
        string oma_code "HUMAN, RATNO, ECOBW"
    }

    GENE {
        int    gene_id PK "Surrogat-Schlüssel"
        int    taxid FK
        int    entrez_id UK "nullable, stammspezifisch bei E. coli"
        string symbol "z.B. TP53, dnaK"
        string systematic_name "z.B. Locus-Tag"
        string uniprot_acc "SwissProt Accession, Brücke E. coli"
        string oma_id "z.B. HUMAN03689"
    }

    GENE_SYNONYM {
        int    gene_id PK, FK
        string synonym PK
    }

    ESSENTIALITY {
        int     gene_id PK, FK "nur humane Gene"
        boolean is_essential
    }

    INTERACTION {
        int    interaction_id PK "BioGRID Interaction ID"
        int    gene_a_id FK
        int    gene_b_id FK
        int    type_id FK
        int    method_id FK
        int    pubmed_id FK
        string throughput "High, Low, Both"
        float  score "nullable"
        string qualifications "nullable, Freitext"
        string source_db "BIOGRID"
    }

    INTERACTION_TYPE {
        int    type_id PK
        string psi_mi_id UK "z.B. MI:2370"
        string name "synthetic lethality, negative genetic, physical association"
        string category "genetic, physical"
    }

    DETECTION_METHOD {
        int    method_id PK
        string psi_mi_id "nullable, z.B. MI:0254"
        string name "z.B. genetic interference"
    }

    PUBLICATION {
        int    pubmed_id PK
        string first_author "z.B. Babu M"
        int    year
    }

    PHENOTYPE {
        string ontology_id PK "z.B. APO:0000112"
        string name "z.B. inviable"
    }

    INTERACTION_PHENOTYPE {
        int    interaction_id PK, FK
        string ontology_id PK, FK
    }

    ORTHOLOGY {
        int    human_gene_id PK, FK
        int    ortholog_gene_id PK, FK
        string rel_type "1:1, 1:n, m:1, m:n"
        int    oma_group "nullable"
    }
```

## Designentscheidungen

- **Ein Gen, mehrere IDs.** Entrez, SwissProt, Symbol und OMA-ID hängen am Gen. Das ist nötig, weil E. coli nur über SwissProt bzw. Symbol mit OMA verknüpft werden kann. Die Entrez-IDs von W3110 (`129xxxxx`) kommen in OMA nicht vor.
- **Eine Tabelle `INTERACTION` für alle Interaktionsarten.** Humane SL-Paare (MI:2370), negative genetische Interaktionen von E. coli (MI:2373, SL-Ersatz) und physische Interaktionen der Ratte unterscheiden sich nur über `INTERACTION_TYPE`.
- **Eine Zeile pro Interaktion und Publikation**, so wie in BioGRID. Wie stark ein Paar belegt ist, ergibt sich aus der Anzahl der Zeilen mit demselben Genpaar.
- **Paare normalisiert speichern.** Beim Import gilt `gene_a_id < gene_b_id`, damit (A,B) und (B,A) nicht doppelt zählen. In der SL-Datei gibt es 36 solcher Duplikate.
- **Mehrwertige Felder aufgelöst.** Synonyme und Phänotypen (in den Quellen `|`-getrennt) liegen in eigenen Tabellen. `Throughput` wird auf `High`, `Low` oder `Both` abgebildet.
- **`ORTHOLOGY` ist gerichtet**, von Mensch zu Ratte oder E. coli, weil die OMA-Dateien immer HUMAN-zu-X-Paare liefern.
- **`ESSENTIALITY` ist optional (1:0..1)**, weil Daten nur für humane Gene vorliegen.
- **Platzhalter-Score.** `score = -20.0000002` (25.670-mal in W3110) wird beim Import als `NULL` behandelt.

## Datei → Tabelle

| Quelldatei | Tabellen |
|---|---|
| `BIOGRID-SYSTEM-Synthetic_Lethality-5.0.263.tab3.txt` | GENE, GENE_SYNONYM, INTERACTION, PUBLICATION, PHENOTYPE, INTERACTION_PHENOTYPE |
| `BIOGRID-ORGANISM-Escherichia_coli_K12_W3110-4.4.225.mitab.txt` | GENE, INTERACTION, PUBLICATION, DETECTION_METHOD |
| `BIOGRID-ORGANISM-Rattus_norvegicus-4.4.225.mitab.txt` | GENE, INTERACTION, PUBLICATION, DETECTION_METHOD |
| `G-HumanEssential.tsv` | ESSENTIALITY, ESSENTIALITY–PUBLICATION |
| `OMA-Pairs_HUMAN-RATNO_*.csv` | GENE (oma_id, uniprot_acc), ORTHOLOGY |
| `OMA-Pairs_HUMAN-ECOBW_*.csv` | GENE (oma_id, uniprot_acc), ORTHOLOGY |

## Kernabfrage (Gen X)

1. In `GENE` das humane Gen X über Symbol oder Entrez-ID suchen.
2. In `INTERACTION` alle Einträge mit Typ *synthetic lethality*, an denen X beteiligt ist → Partner Y.
3. Für jedes Y:
   - `ESSENTIALITY` prüfen. Ist Y essentiell, ist es ein schlechtes Target.
   - Evidenz zählen: Anzahl Publikationen, Throughput.
   - Über `ORTHOLOGY` die Orthologe X' und Y' bestimmen und nachsehen, ob es zwischen ihnen eine Interaktion in Ratte oder E. coli gibt (Erhaltung über die Arten).
