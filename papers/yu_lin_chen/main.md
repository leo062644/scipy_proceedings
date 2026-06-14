---
bibliography:
  - mybib_model_checked.bib
---

# Toward Reliable Localization of Free and Open Source Software: Translation Workflows with LLM Support for QGIS

## Abstract

Localization for free and open source software requires more than fluent text in the target language: interface strings must preserve placeholders, accelerators, markup, entities, newlines, plural behavior, terminology, and structured file semantics. This paper presents a reliability-oriented Python workflow that uses LLMs to support localization of Qt Translation Source (`.ts`) files, using QGIS Traditional Chinese localization as a case study. The workflow treats LLMs served through APIs and LLMs run locally as interchangeable backends. Python splits each source string into `TEXT` spans that contain ordinary language and protected `TOKEN` spans that contain software syntax; the model translates only the text spans, while Python reinserts protected tokens and validates the result before accepting it as a structurally valid candidate.

Ablation experiments on a fixed stratified subset of 3000 segments and complete-corpus C1 experiments on all 28,924 QGIS source messages show the intended trade-off. Here, C1 denotes the full workflow: masked template translation, glossary hints, and three generated candidates per segment. Conditions that send the full string directly to the model without masking can receive lower MQM error rates, especially when glossary hints are supplied, but retain non-zero structural failure rates. The C1 structure-preserving workflow produces zero observed failures under the checked token classes on the complete corpus for all three evaluated backends: Grok, Gemini, and TAIDE. The contribution is therefore not a model leaderboard or a method designed to optimize semantic quality, but a reproducible workflow for making software localization with LLMs safer and auditable.

---

## 1. Introduction

Localization quality is a practical barrier to adoption for scientific and geospatial software. QGIS is a widely used free and open-source geographic information system [@qgis_software; @graser2025_qgis], but the usability of a localized interface depends not only on translation coverage. It also depends on linguistic adequacy, terminology consistency, and preservation of software-specific formatting constraints. For Traditional Chinese users, untranslated strings, residual English, inconsistent GIS terminology, and broken placeholders or markup can increase the learning barrier and reduce trust in the software.

LLMs can assist localization, but software localization is not ordinary document translation. A Qt `.ts` file is an XML-based localization resource containing source strings, translations, contexts, locations, message states, plural forms, and translator metadata. User-visible strings may contain runtime placeholders such as `%1` and `%n`, brace placeholders such as `{0}` or `{feature_id}`, keyboard accelerators such as `&File`, HTML-like markup, escaped entities, numeric tokens, and significant whitespace. A translation that appears fluent may still be unsuitable for deployment if it drops `%1`, changes `<b>...</b>`, removes a newline, or mistranslates a domain term.

This paper presents a validation-first workflow for QGIS localization with LLM support. By validation-first, this paper means that generated translations are not trusted simply because they are model outputs. A candidate is accepted only after Python checks XML well-formedness and software-format invariants; rows that do not pass blocking checks are logged for human review instead of being silently treated as completed translations. The central claim is not that a single LLM can automatically replace translators. Instead, LLMs should be placed inside a reviewable Python workflow: deterministic validation protects software structure, project glossaries provide terminology guidance, LLM judges scale semantic review, and human reviewers retain final responsibility.

The workflow is intentionally not tied to one model backend. Translation can be performed through API models or local models. This matters for FOSS communities because contributors may prefer local inference for cost, privacy, offline processing, or reproducibility, while others may use cloud APIs for stronger translation or support for structured outputs. The paper therefore evaluates a workflow across backends, not a single provider.

**Contributions.** This paper makes four contributions:

1. A compact validation-first Python workflow for QGIS `.ts` localization with LLM support.
2. A structure-preserving template translation method in which LLMs translate only ordinary language spans while Python deterministically preserves and reinserts software-format tokens.
3. A shared backend interface for both API-based and local LLMs, enabling the same workflow to run with Gemini, Grok, and a local TAIDE-family model.
4. A reproducible evaluation protocol using fixed subsets, deterministic structural checks, terminology diagnostics, production runs on the complete corpus, and sampled semantic evaluation using MQM.

---

## 2. Background and Related Work

### 2.1 Software localization as structured data

Qt `.ts` files are not plain text translation memories. The Qt Linguist TS format is an XML-based file format described by an XSD specification, and its message records may contain message type, source text, translation text, context, locations, comments, and plural forms [@qt_ts_format]. QGIS translation workflows use Transifex and Qt Linguist-style resources for desktop application translation [@qgis_translation_guidelines]. This makes `.ts` localization closer to a transformation of structured data with explicit constraints than to unconstrained natural-language translation.

In this setting, a localization workflow should distinguish between:

```text
Human-language content: text that should be translated.
Software-format content: tokens that must be preserved exactly.
```

A generic LLM prompt that treats the source as plain text may produce fluent translations while silently damaging software-format content. This motivates translation that is aware of protected tokens, deterministic validation, and explicit reporting. Localization engineering and computer-assisted translation literature similarly treats localization as a managed workflow involving software resources, terminology, translation memory, testing, and quality assurance rather than a single act of sentence translation [@esselink_2000; @bowker_2002]. This paper follows that workflow-oriented view, but makes the structural QA layer executable and reproducible in Python.

### 2.2 QGIS terminology and Traditional Chinese localization

QGIS localization is domain-specific. Terms such as `layer`, `feature`, `raster`, `vector`, `CRS`, `geometry`, `attribute table`, and `processing algorithm` must be handled consistently across menus, processing tools, database modules, and error messages. Generic LLM translation can produce fluent output while drifting across terminology variants.

The workflow uses project glossaries as reference hints. Glossary entries are not used as mechanical replacement rules because GIS terms often appear in derived or compound forms. For example, `raster`, `raster layer`, `rasterize`, and `rasterization` may require different Traditional Chinese expressions. Therefore, the workflow retrieves glossary hints, includes them in the prompt, and validates output after generation rather than forcing direct string substitution.

### 2.3 Structured generation, token preservation, and semantic evaluation

Methods for structured output can reduce malformed model responses by asking the model to return a machine-readable object such as JSON [@google_structured_outputs; @xai_structured_outputs]. In software localization, however, valid JSON is not enough: placeholders, markup, accelerator markers, and whitespace remain software-format invariants that must be preserved. This workflow therefore uses structured JSON output only for translated text spans. Protected tokens are not generated by the model at all; they are reinserted by deterministic Python code. In this paper, the phrase structured generation refers to the JSON text-span response, while token preservation refers to the separate deterministic layer.

Semantic quality is evaluated separately. MQM is an analytic translation quality evaluation framework applicable to human translation, machine translation, and AI-generated translation [@mqm_what_is; @lommel_mqm_2014]. @kocmi_federmann_2023 introduced GEMBA-MQM, an LLM-based method for detecting translation quality error spans using a fixed few-shot prompt and an MQM-style rubric. In this paper, LLM as judge is used as scalable assisted annotation, not as ground truth. Model and condition labels are hidden from the judge, and deterministic structural results are reported separately from semantic MQM scores.

---

## 3. Method

### 3.1 Workflow overview and model backends

The workflow has six stages:

1. Parse the Qt `.ts` file into translation units.
2. Split each source string into protected tokens and translatable text spans.
3. Retrieve glossary hints from tabular terminology resources for the source segment.
4. Ask an interchangeable LLM backend to translate only the text spans.
5. Reinsert protected tokens deterministically and select a structurally valid candidate.
6. Validate the generated `.ts` file and emit structural, deterministic QA, and MQM reports.

```{figure} workflow.svg
:name: fig-workflow-overview
:alt: Workflow for Qt .ts localization
:width: 100%

Workflow for Qt `.ts` localization. Stages 2--5 repeat for each translation unit; protected tokens are reinserted before a structurally valid candidate is selected.
```

Figure 1 separates deterministic processing from model-dependent generation. Parsing, token splitting, glossary lookup, token reinsertion, candidate selection, and validation are implemented once in Python and are shared across all backends. The backend adapter is the only part that changes when the workflow calls Gemini, Grok, or a local TAIDE model. This design makes the comparison focus on whether the same workflow remains reliable across different model access modes, rather than on provider-specific code.

The deterministic and glossary layers are applied uniformly across all model backends. The only backend-specific component is the adapter that converts a common request object into an API call or a local inference call.

**Table 1. Model backends.**

| Backend | Access | Model ID | Role |
|---|---|---|---|
| Gemini | API | `gemini-3.1-flash-lite` | Translation backend |
| Grok | API | `grok-4.3` | Translation backend and MQM judge |
| TAIDE | Local | `taide/Gemma-3-TAIDE-12b-Chat-2602` | Local translation backend |

Model-version details are kept out of the main table to reduce width. Gemini and Grok are API backends with support for structured outputs [@google_gemini_flash_lite_api; @google_structured_outputs; @xai_grok_43; @xai_structured_outputs]. The TAIDE backend is a Hugging Face Hub checkpoint whose model card states that it is based on `google/gemma-3-12b-pt`; the base Gemma 3 12B pre-trained checkpoint and Gemma 3 technical report provide the relevant Gemma 3 model context [@taide_gemma3_12b_2602; @google_gemma3_12b_pt; @gemma3_technical_report]. Because API model aliases and model repositories are dynamic, the artifact records API request dates and prompt versions, as well as the local checkpoint revision, download date, license terms, inference engine, quantization, GPU, VRAM, and decoding settings.

### 3.2 Translation unit and token sequence model

Each Qt message is represented as a translation unit:

$$
u = (id, i, c, s, t, m, l, a),
$$

where $id$ is a stable identifier, $i$ is the message index, $c$ is the Qt context, $s$ is the source string, $t$ is the target translation, $m$ contains message metadata, $l$ contains source-code locations, and $a$ contains extracted artifacts such as placeholders, tags, entities, numeric tokens, accelerator markers, and newline tokens.

For a source string $s$, the splitter produces an alternating sequence of text spans and protected tokens:

$$
s = x_0 \tau_1 x_1 \tau_2 \cdots \tau_k x_k,
$$

where $x_j$ are translatable or copyable text spans and $\tau_j$ are protected software-format tokens. The protected token sequence is:

$$
T(s) = (\tau_1, \tau_2, \ldots, \tau_k).
$$

For any accepted output $\hat{t}$, the structural preservation condition for token classes checked by the validator is:

$$
T_r(s) = T_r(\hat{t}) \quad \forall r \in \mathcal{R},
$$

where $\mathcal{R}$ is the set of token extractors, such as Qt placeholders, brace placeholders, printf placeholders, HTML/XML tags, entities, numeric tokens, newlines, and accelerators.

A complete example makes the representation concrete. Suppose the source message is shown as follows, with the newline escaped as `\n` for readability:

```xml
<message>
  <location filename="src/app/qgsattributedialog.cpp" line="125"/>
  <source>Open &amp;Attribute Table\nLayer: %1, feature ID: {feature_id}</source>
  <translation type="unfinished"></translation>
</message>
```

The corresponding translation unit can be represented as:

```text
u = (
  id = "QgsAttributeDialog:125",
  i  = 125,
  c  = "QgsAttributeDialog",
  s  = "Open &Attribute Table\nLayer: %1, feature ID: {feature_id}",
  t  = "",
  m  = {translation_type: "unfinished"},
  l  = [("src/app/qgsattributedialog.cpp", 125)],
  a  = {
        accelerators: ["&"],
        newlines: ["\n"],
        qt_placeholders: ["%1"],
        brace_placeholders: ["{feature_id}"]
       }
)
```

The splitter then produces this alternating sequence:

```text
x0   = "Open "
tau1 = "&"
x1   = "Attribute Table"
tau2 = "\n"
x2   = "Layer: "
tau3 = "%1"
x3   = ", feature ID: "
tau4 = "{feature_id}"
```

The protected token sequence is therefore:

```text
T(s) = ("&", "\n", "%1", "{feature_id}")
```

The model is asked to translate only the text spans and may return JSON such as:

```json
{
  "T0": "開啟",
  "T1": "屬性表",
  "T2": "圖層：",
  "T3": "，圖徵 ID："
}
```

Python then reinserts the protected tokens in their original order:

```text
開啟&屬性表
圖層：%1，圖徵 ID：{feature_id}
```

The accepted output still has the same checked token sequence:

```text
T(accepted output) = ("&", "\n", "%1", "{feature_id}")
```

This example also shows the boundary of the model's responsibility. The model chooses the Chinese wording for `Open`, `Attribute Table`, `Layer:`, and `Feature ID:`, but it does not need to regenerate the accelerator marker, placeholder, newline, or brace placeholder.

### 3.3 Structure-preserving template translation

Each source string is split into `TEXT` parts and protected `TOKEN` parts. Protected `TOKEN` parts include:

```text
Qt placeholders:        %1, %2, %n, %L1
Brace placeholders:     {0}, {feature_id}
printf placeholders:    %s, %d, %.2f
HTML/XML tags:          <b>, </b>, <p align="right">
Entities:               &amp;, &lt;, &gt;, &nbsp;
Escaped controls:       \n, \t, \r
Actual controls:        newline, tab, carriage return
Keyboard accelerators:  &, &&
Numeric literals:       100, 3.44, -1
```

The LLM is not asked to reproduce these tokens. It translates only the ordinary language spans and returns JSON keyed by text-part identifiers. Python then reinserts the protected tokens exactly.

Example source:

```xml
<p align="right">Algorithm author: {0}</p>
```

Template parts:

```text
TOKEN: <p align="right">
TEXT : Algorithm author:
TOKEN: {0}
TOKEN: </p>
```

Model response:

```json
{"T0": "演算法作者："}
```

Reassembled output:

```xml
<p align="right">演算法作者：{0}</p>
```

This design makes preservation of checked token classes deterministic for accepted candidates. The guarantee boundary is limited: the workflow can preserve defined software-format invariants, but it cannot guarantee semantic adequacy, fluency in the target language, or UI usability.

### 3.4 Glossary retrieval from tabular resources

Project glossaries are provided as tabular terminology resources. The experiments use spreadsheet-style files, but the method only assumes that each resource can be normalized into records containing:

```text
source_term, target_terms, forbidden_terms, priority, domain, note
```

To avoid scanning every glossary entry for every segment, source terms are tokenized, normalized, and indexed by n-gram keys:

$$
I[n, g] = \{e \in G : \operatorname{key}_n(e.source\_term) = g\},
$$

where $G$ is the glossary and $g$ is a normalized n-gram key. For a source segment $s$, the retrieved hint set is:

$$
H(s) = \operatorname{TopK}\left(\bigcup_{n=1}^{N_{max}} I[n, \operatorname{ngrams}_n(s)], K\right).
$$

Glossary hints are passed to the LLM as contextual guidance, not mechanical replacements. This allows the model to adapt terminology to context while keeping prompt size controlled.

### 3.5 Candidate selection and non-accepted rows

Each segment may produce multiple candidate translations. The default production condition uses three candidates. Python reassembles each candidate, validates it, and selects the best valid candidate by deterministic quality-risk score:

$$
Q(\hat{t}) = \sum_{e \in E(\hat{t})} w(severity(e)) + \lambda_1 R_{latin}(\hat{t}) + \lambda_2 R_{len}(s,\hat{t}).
$$

The accepted candidate is:

$$
j^* = \arg\min_{j : E_{blocking}(\hat{t}_{u,j})=\emptyset} Q(\hat{t}_{u,j}).
$$

If no generated candidate passes strict validation, the row is not treated as accepted localized content and is logged for review. The main paper does not use this review state as a headline quality metric because it combines several operational causes: no valid candidate, backend exception, malformed JSON, or intentionally conservative rejection. Detailed review-state counts are reported in the artifact. The main text focuses on structural failure rate, deterministic content/completion diagnostics, and MQM semantic quality.

### 3.6 Guarantee boundary

The guarantee boundary is intentionally narrow:

- The workflow preserves checked token classes for accepted masked-template outputs.
- It eliminates observed structural failures only under the implemented extractors.
- It does not prove semantic adequacy, fluency, terminology correctness, UI usability, or cultural appropriateness.

If a token class is not defined by the extractor, it is outside the guarantee boundary. If a segment is logged for review rather than accepted as a structurally valid candidate, it should not be counted as completed user-facing localization. If a result has zero observed structure failures on a finite subset or corpus, this should be reported as zero observed failures under checked token classes, not as proof of a true 0% population error rate.

### 3.7 Workflow procedure

The procedure is summarized below in MyST-friendly pseudocode. Full implementation details are in the repository.

```text
Input:  Qt TS file S, glossaries G, backend b, condition C
Output: translated TS file, translation log, deterministic reports

U <- parse_ts(S)
I <- build_glossary_index(G)

for each translation unit u in U:
    X, T <- split_text_and_tokens(u.source)
    H    <- retrieve_glossary_hints(I, u.source, top_k)
    Y    <- generate_candidates(b, u, X, T, H, C)
    V    <- empty list

    for each candidate y in Y:
        t_hat  <- reinsert_tokens(y, T)
        issues <- validate(u.source, t_hat)
        if no blocking issue is found:
            add (t_hat, deterministic_risk_score(t_hat)) to V

    if V is not empty:
        write the lowest-risk structurally valid candidate as accepted output
    else:
        write a review-required row and log diagnostics

validate XML well-formedness
emit CSV, JSON, and Markdown reports
```

---

## 4. Implementation and Experiment Conditions

The implementation is a Python command-line workflow. The main runner builds fixed subsets, runs C0--C4 ablation conditions, writes generated `.ts` files, and calls evaluation and scoring scripts. The same selected subset is reused across backends to make paired comparisons possible.

**Table 2. Ablation conditions.**

| Condition | Short name | Masking | Glossary hints | Candidates | Purpose |
|---|---|---:|---:|---:|---|
| C0 | Direct baseline | no | no | 1 | Direct translation without masking |
| C1 | Complete workflow | yes | yes | 3 | Production condition |
| C2 | Direct + glossary | no | yes | 3 | Tests glossary/candidates without masking |
| C3 | Masked without glossary | yes | no | 3 | Tests masking without glossary hints |
| C4 | Single candidate | yes | yes | 1 | Tests the value of generating multiple candidates |

C0--C4 are ablations of one workflow rather than five unrelated methods. C0 and C2 are conditions without masking; C1, C3, and C4 are masked conditions. Evaluation on the complete corpus focuses on C1 because it is the pre-specified production workflow. Scaling all ablation conditions to the complete corpus would primarily increase cost and review burden rather than change the deployment question: whether the selected production workflow remains structurally safe at QGIS scale.

### 4.1 Direct generation and residual risk

A natural question is whether stronger prompting is enough. C0 and C2 should be interpreted conservatively: they demonstrate residual risk under the implemented direct prompts, but they are not an exhaustive benchmark of every possible prompt-only or constrained-decoding design. In these conditions without masking, the model sees the full source string and remains responsible for reproducing protected software-format tokens. C2 additionally receives glossary hints and three candidates. These settings can improve semantic quality, but they still leave protected-token preservation inside the model generation task. Masked conditions change the problem decomposition: the LLM translates language spans, and deterministic code preserves software-format spans.

### 4.2 Validation fixtures and smoke tests

The artifact exercises the deterministic layer through a bundled mini deterministic evaluation fixture. The fixture contains archived 100-segment C0--C4 `.ts` outputs, condition metadata, selected-segment metadata, and evaluation outputs. The default no-key reproduction reruns the deterministic evaluator and scoring scripts on these archived `.ts` files, so reviewers can verify deterministic token checks, accelerator handling, XML well-formedness checks, condition comparison, scoring behavior, MQM request planning, and table generation without calling any model API.

The fixture is not presented as a full regeneration of the paper experiments: it does not rerun LLM translation, candidate generation, glossary-assisted prompting, or token reassembly during generation. Those steps are covered by the workflow scripts and can be exercised through optional API/local reruns, while the default no-key path provides a stable offline check of the deterministic evaluation and reporting code path.

---

## 5. Evaluation as Method Validation

The evaluation validates the workflow rather than ranking models. It has two reported layers and one complementary semantic-review protocol.

1. **Ablation subset:** C0--C4 on the same 3000-segment stratified subset for Grok, Gemini, and TAIDE.
2. **Complete corpus:** C1 complete workflow on all 28,924 QGIS source messages for each backend.
3. **Semantic review:** MQM-style sampled judging evaluates semantic adequacy, terminology, fluency, and locale style on the ablation outputs, while remaining separate from deterministic structure scoring.

### 5.1 Structural metrics

For condition $c$ and $N$ checked segments, the structure-failure rate is:

$$
\hat{p}_c = \frac{1}{N}\sum_{u=1}^N \mathbb{1}\left[E_{struct}(u,c) \neq \emptyset\right].
$$

For zero observed failures, the one-sided approximate 95% upper bound is reported using the rule of three [@hanley_lippmanhand_1983]:

$$
p_{upper} \approx \frac{3}{N}.
$$

For the 3000-segment ablation subset, this corresponds to approximately 0.1000%. For the 28,924-message complete corpus, it corresponds to approximately 0.0104%.

### 5.2 Deterministic content/completion QA metrics

The term **deterministic content/completion QA** is used consistently for possibly untranslated text, high English residue, empty outputs, missing translation elements, rows with no valid candidates, glossary target missing counts, and forbidden term counts. These metrics are engineering diagnostics, not a complete translation-quality score. A condition can improve structural safety while lowering deterministic content/completion QA if it creates more review-required or untranslated-looking rows.

### 5.3 MQM-style semantic review metrics

Semantic quality is evaluated with MQM-style scoring. For segment $u$, let $P_u$ be the set of MQM penalties assigned by the judge. The weighted MQM error rate is:

$$
MQM\text{-}ER = 1000 \times \frac{\sum_{u=1}^N\sum_{p \in P_u} p}{\sum_{u=1}^N |s_u|}.
$$

Lower values indicate fewer weighted semantic errors per 1000 source characters. The auxiliary MQM score on a 0--100 scale is reported for readability; the error rate is the main semantic metric.

The MQM analysis is not framed as a claim that masking should improve semantic quality. The trade-off question is whether the semantic cost of masking is small enough to justify the structural safety gain. Let:

$$
\Delta_{MQM} = MQM\text{-}ER(C1_{masked}) - MQM\text{-}ER(C0_{direct}).
$$

A value near zero indicates no material semantic penalty; a small positive value indicates a modest semantic cost; a negative value indicates that the masked workflow also improves semantic quality. In all cases, MQM is interpreted separately from structural failure rate.

---

## 6. Results

### 6.1 Ablation subset: structural safety and semantic trade-off

The full C0--C4 ablation outputs are archived in the artifact. To keep the main paper compact, Table 3 reports the four conditions needed for the main trade-off comparison: C0 direct baseline, C1 complete workflow, C2 direct generation with glossary hints, and C4 single-candidate masked workflow. C3 is omitted from the compact table because it is not needed for the main trade-off comparison; it is included in the artifact and shows the same zero-observed-structure-failure pattern as the other masked conditions.

**Table 3. Compact ablation summary on the 3000-segment subset.** `Det. QA` abbreviates deterministic content/completion QA, an engineering diagnostic rather than a complete translation-quality score.

| Backend | Cond. | Short name | Structure failed % ↓ | Det. QA ↑ | MQM-ER ↓ |
|---|---|---|---:|---:|---:|
| Grok 4.3 | C0 | Direct baseline | 5.67 | 93.60 | 6.289 ± 0.368 |
| Grok 4.3 | C1 | Complete workflow | 0.00 | 92.57 | 9.005 ± 2.807 |
| Grok 4.3 | C2 | Direct + glossary | 5.13 | 94.54 | 3.630 ± 1.488 |
| Grok 4.3 | C4 | Single candidate | 0.00 | 92.40 | 11.016 ± 2.372 |
| Gemini 3.1 Flash-Lite | C0 | Direct baseline | 8.70 | 92.72 | 7.194 ± 1.702 |
| Gemini 3.1 Flash-Lite | C1 | Complete workflow | 0.00 | 92.97 | 11.109 ± 2.481 |
| Gemini 3.1 Flash-Lite | C2 | Direct + glossary | 9.30 | 92.81 | 4.904 ± 1.945 |
| Gemini 3.1 Flash-Lite | C4 | Single candidate | 0.00 | 92.82 | 12.482 ± 3.276 |
| TAIDE 12B | C0 | Direct baseline | 30.10 | 76.11 | 37.339 ± 10.414 |
| TAIDE 12B | C1 | Complete workflow | 0.00 | 84.52 | 40.736 ± 6.212 |
| TAIDE 12B | C2 | Direct + glossary | 25.17 | 79.63 | 31.564 ± 5.943 |
| TAIDE 12B | C4 | Single candidate | 0.00 | 80.72 | 41.402 ± 6.335 |

The main observation is structural rather than semantic. Under the implemented conditions without masking, C0 and C2 retain non-zero structural risk for every backend: Grok C0/C2 have 5.67% and 5.13% structure-failed rates, Gemini C0/C2 have 8.70% and 9.30%, and TAIDE C0/C2 have 30.10% and 25.17%. These results do not claim to rule out all possible prompt-only methods; they show that direct generation leaves a residual token-preservation risk in this workflow. In contrast, all masked conditions produce zero observed structure failures under the checked token classes. The full item-level breakdown shows that direct generation failures concentrate in newline, accelerator, HTML/XML tag, and placeholder preservation; those details are provided in the artifact rather than reproduced as a large table in the main text.

MQM shows the expected semantic trade-off. C2, the direct condition with glossary hints, obtains the lowest MQM error rate for all three backends, but it also retains non-zero structural failure rates. C1 has a higher MQM error rate than C0 and C2, especially for API models, but removes the observed structural failures under the implemented extractors. This supports the paper's main distinction: direct generation can be more fluent or semantically preferred, while masked settings are safer for deployment.

### 6.2 Effect of generating multiple candidates

C1 and C4 isolate the value of generating multiple candidates. Both use masking and glossary hints from tabular resources; C1 uses three candidates and C4 uses one. Both achieve zero observed structure failures, so candidate count is not the source of structural safety. Its value is secondary: it gives deterministic validation more candidate choices.

For API backends, the effect is small. Relative to C4, C1 improves deterministic content/completion QA by +0.17 for Grok and +0.15 for Gemini. For TAIDE, the effect is larger: deterministic content/completion QA improves by +3.80, and the average valid candidate count increases from 0.689 to 2.119. MQM also favors C1 over C4 for all three backends, but the differences are modest relative to repeated-judge variability. We therefore interpret three-candidate selection as a robustness mechanism, not as the primary safety contribution. The primary safety mechanism remains masking plus deterministic reassembly.

### 6.3 Complete-corpus C1 production results

Complete-corpus C1 experiments were run on all 28,924 QGIS source messages for each backend. C1 is the production workflow: masking, glossary hints from tabular resources, and three candidates.

**Table 4. C1 production-condition comparison on the complete corpus.**

| Backend | Messages checked | Structure failed % ↓ | Structure score ↑ | Det. QA ↑ | Avg. valid candidates | Possibly untranslated |
|---|---:|---:|---:|---:|---:|---:|
| Grok 4.3 | 28,924 | 0.00 | 100.000 | 92.11 | 2.938 | 1,394 (4.82%) |
| Gemini 3.1 Flash-Lite | 28,924 | 0.00 | 100.000 | 90.09 | 2.935 | 1,825 (6.31%) |
| TAIDE 12B | 28,924 | 0.00 | 100.000 | 65.13 | 2.052 | 6,674 (23.07%) |

The complete corpus result confirms that the subset finding scales to the complete QGIS source file: all three backends produce zero observed structural failures under the checked token classes in C1. The local TAIDE backend is structurally safe under the workflow, but its deterministic content/completion QA is much lower. This indicates that the local backend is a viable offline deployment path for structural preservation, while currently requiring substantially more review than the API backends.

---

## 7. Reproducibility and Artifact Availability

The accompanying repository is available at <https://github.com/wendy062644/qgis-llm-localization-workflow>. A versioned release tag will be archived for the final proceedings version. The artifact is organized around three reproducibility levels so that reviewers can distinguish no-key deterministic checks from costlier model reruns.

**Table 5. Reproducibility levels.**

| Level | Command or location | New model calls? | What it verifies |
|---|---|---:|---|
| Mini deterministic evaluation | `python scripts/run_repro.py full-mini`; `experiments/demo_ablation_grok_100/` | No | Reruns deterministic evaluation, scoring, condition comparison, and MQM request generation on bundled 100-segment C0--C4 `.ts` outputs. |
| Extended archived-output scoring | `python scripts/run_repro.py score --experiment <experiment> --force-eval` | No | Uses a downloaded paper-experiment directory to recompute structure and deterministic content/completion metrics from archived generated `.ts` files rather than from preformatted table rows. |
| Optional MQM or translation rerun | `python scripts/run_repro.py mqm ... --run-grok`; `python scripts/run_repro.py translate ...` | Yes | Rebuilds MQM judge outputs or regenerates a small translation smoke run using API credentials or local inference. |

The default mini deterministic evaluation is reviewer-friendly and offline. It uses the same C0--C4 condition definitions and the same deterministic evaluator/scorer as the paper experiments, but on a smaller 100-segment archived-output fixture so that it can run quickly. It should be interpreted as an executable validation of the deterministic evaluation, scoring, comparison, and request-planning code paths, not as a full regeneration of LLM translation outputs or as a substitute for the 3000-segment and 28,924-message experimental artifacts.

The compact release contains the mini fixture, source `.ts` file, tabular glossary resources used in the experiments, C0--C4 configuration files, workflow scripts, archived demo `.ts` outputs, workflow manifests, deterministic evaluation outputs, MQM request-planning outputs, and an `.env.example` file. The final proceedings artifact includes paper-facing table artifacts for the reported 3000-segment ablation, complete-corpus C1 comparison, and MQM summaries. Full paper-scale archived-output recomputation is provided through a larger extended experiment archive containing the generated `.ts` outputs for the 3000-segment ablation and 28,924-message C1 runs. With that archive, the same scoring command recomputes deterministic metrics from archived outputs without new model calls. Rerunning translation or MQM judging requires the appropriate API credentials or local inference hardware. API keys are excluded from the repository and are supplied through environment variables such as `XAI_API_KEY` and `GEMINI_API_KEY`.

The repository README documents the quickstart, deterministic scoring, MQM request planning, optional MQM judge run, and optional API translation smoke run.

---

## 8. Discussion and Limitations

The workflow improves reliability by separating structural safety, terminology guidance, semantic evaluation, and human review. The deterministic layer is the most objective because it checks software artifacts that should be preserved. The structure-preserving template layer strengthens this idea by removing protected tokens from the LLM generation space. Generating multiple candidates is a secondary robustness layer; it improves candidate availability, especially for the local backend, but it is not the mechanism that creates the structural guarantee.

The role of masking should be interpreted carefully. Masking does not solve semantic adequacy by itself, and the MQM results show a modest semantic cost for C1 compared with C0 and C2, the two conditions without masking. However, this is a different risk category from interface breakage. C0/C2 are evidence of residual risk under the implemented direct generation settings, not an exhaustive dismissal of all prompt-only approaches. C2 can be semantically stronger, but it still leaves non-zero structural failure rates in this experiment. In practical terms, masking trades a measured semantic cost for a much stronger guarantee that the localized interface remains structurally usable under the checked token classes.

The local LLM results should be read as evidence for deployability, not model superiority. TAIDE achieves the same zero observed structural failure result under C1, but its deterministic content/completion QA and review-workload indicators are much worse than the API backends. In the archived status diagnostics, 7,265 of 28,924 TAIDE C1 rows, or 25.12%, entered a review-required source-fallback state. This is not a fatal flaw for the workflow, because the workflow prevents those rows from being silently accepted as structurally valid localized content. It does mean that the local backend currently requires substantially more human review before release; in the present configuration, TAIDE should be treated as an offline deployment path with a larger review queue rather than as a drop-in substitute for the API backends.

The glossary layer depends on glossary quality. Because glossary matches are hints rather than replacements, the workflow avoids many unnatural literal substitutions, but terminology correctness still requires evaluation.

Semantic review is necessary because the deterministic layer does not evaluate adequacy, fluency, domain correctness, or target-locale style. LLM judges can be useful for scalable review, but they are assisted annotation rather than ground truth. Human review remains necessary for release-critical strings, ambiguous GIS terms, and user-facing messages with high impact.

### 8.1 Practical deployment recommendation

For a FOSS localization team, the recommended deployment workflow is:

1. Run the selected production condition.
2. Accept only rows with zero blocking structural issues.
3. Treat review-required rows as incomplete, not as translated content.
4. Prioritize review by English residue, terminology warnings, semantic severity, and review status.
5. Run XML well-formedness and deterministic validation before each release candidate.
6. Use human review for final acceptance of UI-critical strings and domain terminology.

The workflow preserves checked token classes and creates auditable review queues; it does not eliminate all localization errors or prove translation quality.

---

## 9. Conclusion

This paper presents a Python workflow for reliable localization of QGIS Qt `.ts` files with LLM support, without tying the method to a single model backend. The workflow supports both API-based and local LLMs, uses tabular glossary resources as contextual terminology hints, and applies structure-preserving template translation to preserve software-format tokens deterministically.

Ablation results on the 3000-segment subset show that conditions without masking produce non-zero structural failure rates for Grok, Gemini, and TAIDE, while masked template conditions produce zero observed structure failures under checked token classes for all three backends. Complete-corpus C1 results extend this finding to all 28,924 QGIS source messages: Grok, Gemini, and TAIDE all have zero observed structure failures under the implemented extractors. Semantic results using MQM show that masking is not a semantic optimizer: C1 has a modestly higher MQM error rate than the direct baselines, and C2 has the best semantic scores but remains structurally unsafe. C1 versus C4 further shows that generating three candidates helps robustness, especially for TAIDE, but the primary safety mechanism remains masking plus deterministic reassembly.

The broader contribution is methodological. Reliable FOSS localization should not depend on raw LLM output alone. It should be built as a reproducible workflow in which deterministic validation protects software structure, glossary retrieval supports terminology consistency, MQM evaluation with LLM assistance scales semantic review, and human reviewers provide final credibility.

---

## Acknowledgements and AI Disclosure

Generative AI was used for outlining, wording, and formatting assistance. The authors verified the technical claims, references, experimental results, source code, and final manuscript text. LLM outputs used for translation or MQM-style judging are treated as assisted annotations, not ground truth.

---

## References

```{bibliography}
```
