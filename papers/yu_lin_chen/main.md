---
bibliography:
  - mybib.bib
---

# Toward Reliable Localization of Free and Open Source Software: Translation Workflows with LLM Support for QGIS

## Abstract

Localization for free and open source software requires more than fluent target-language text. In Qt `.ts` files, translations must remain valid software resources: runtime placeholders, markup, escaped entities, line breaks, plural forms, terminology, XML structure, and message metadata must be preserved or handled according to explicit rules. This paper presents a Python workflow that uses LLMs to support localization of Qt Translation Source (`.ts`) files, using QGIS Traditional Chinese localization as a case study. A Python script segments each source string into translatable human-language spans and protected format-control tokens such as placeholders, markup, entities, and line breaks. The LLM translates only the human-language spans, while Python reinserts the protected tokens and validates the reassembled result. The workflow supports both cloud-based and local LLMs.

Condition-comparison experiments on a fixed stratified subset of 3,000 segments and complete-corpus experiments on all 28,924 QGIS source messages demonstrate the intended trade-off. Direct-generation settings that send the full source string to the model, including variants with glossary hints and multiple generated candidates, can achieve lower MQM error rates but still suffer from non-zero structural failure rates. In contrast, the selected production workflow—masked template translation with glossary hints and three generated candidates per segment—produces zero observed failures under the checked token classes on the complete corpus for all three evaluated backends: Grok, Gemini, and TAIDE. This work therefore contributes not a model leaderboard or a method designed to optimize semantic quality, but a reproducible workflow for making software localization with LLMs safer and auditable.

---

## 1. Introduction

Localization quality is a practical barrier to the adoption of scientific software. A localized interface must do more than translate many strings: it must use clear language, keep terms consistent, and preserve software formatting rules. Taking QGIS, a widely used free and open-source geographic information system [@qgis_software; @graser2025_qgis], as an example, untranslated strings, residual English, inconsistent GIS terms, and broken placeholders or markup can make the interface harder to learn and reduce users' trust in the software.

LLMs can assist localization, but software localization is not merely document translation. A Qt `.ts` file is an XML-based localization resource containing source strings, translations, contexts, locations, message states, plural forms, and translator metadata. User-visible strings may contain runtime placeholders such as `%1` and `%n`, brace placeholders such as `{0}` or `{feature_id}`, Qt mnemonic markers for Alt-key shortcuts such as `&File`, HTML-like markup, escaped entities, numeric tokens, and significant whitespace. A translation that looks acceptable in plain-text review may still be release-blocking if it violates format constraints—for example, by omitting `%1`, altering `<b>...</b>`, removing a required line break, or using an incorrect GIS term.

This paper presents a validation-first workflow for QGIS localization with LLM support, where generated translations are not trusted simply because they are model outputs. A translation is accepted only if Python confirms that the reassembled `.ts` message is well-formed XML and passes all blocking format-preservation checks; otherwise, the row is logged for human review rather than silently treated as completed localization. The central claim is not that a single LLM can automatically replace translators. Instead, LLMs should be placed inside a reviewable Python workflow: rule-based validation protects software structure, project glossaries provide terminology guidance, LLM judges scale semantic review, and human reviewers retain final responsibility.

The workflow is intentionally not tied to one model backend. Translation can be performed through cloud models or local models. This matters for FOSS communities because contributors may prefer local inference for cost, privacy, offline processing, or reproducibility, while others may use cloud models for stronger translation quality or better support for JSON-style structured responses. The paper therefore evaluates a workflow across models.

**Contributions.** This paper makes four contributions:

1. A compact validation-first Python workflow for QGIS `.ts` localization with LLM support.
2. A structure-preserving template translation method in which LLMs translate only human-language spans while Python preserves protected format-control tokens and puts them back by rule.
3. A shared backend interface for both cloud-based and local LLMs, enabling the same workflow to run with Gemini, Grok, and a local TAIDE-family model.
4. A reproducible evaluation protocol using fixed subsets, rule-based structural checks, terminology diagnostics, production runs on the complete corpus, and sampled semantic evaluation using MQM.

---

## 2. Background and Related Work

### 2.1 Software localization as structured data

The Qt Linguist TS format (`.ts`) is not a plain-text translation memory. Instead, it is an XML-based file format described by an XSD specification, and its message records may contain message type, source text, translation text, context, locations, comments, and plural forms [@qt_ts_format]. This makes `.ts` localization closer to a transformation of structured data with explicit constraints than to unconstrained natural-language translation.

In this setting, a localization workflow should distinguish between:

```text
Human-language content: words and phrases intended for translation.
Formatting-language content: string tokens that must be preserved exactly.
```

A generic LLM prompt that treats the source as plain text may produce readable translations while silently damaging format-control tokens. This motivates a translation workflow that is aware of protected tokens, rule-based validation, and explicit reporting. Localization engineering and computer-assisted translation literature similarly treats localization as a managed workflow involving software resources, terminology, translation memory, testing, and quality assurance (QA) rather than a single act of sentence translation [@esselink_2000; @bowker_2002]. This paper follows that workflow-oriented view, but makes the structural QA layer executable and reproducible in Python.

### 2.2 QGIS terminology and Traditional Chinese localization

QGIS localization is domain-specific. Terms such as `layer`, `feature`, `raster`, `vector`, `CRS`, `geometry`, `attribute table`, and `processing algorithm` must be handled consistently across menus, processing tools, database modules, and error messages. Generic LLM translation can produce fluent output while drifting across terminology variants.

To reduce this drift, the workflow uses a glossary-assisted retrieval approach, in which relevant entries from a GIS glossary are retrieved as reference hints for generation. Glossary entries are not used as mechanical replacement rules because GIS terms often appear in derived or compound forms. For example, `raster`, `raster layer`, `rasterize`, and `rasterization` may require different Traditional Chinese expressions. Therefore, the workflow retrieves glossary hints, includes them in the prompt, and validates output after generation rather than enforcing direct string substitution.

### 2.3 Structured generation, token preservation, and semantic evaluation

Methods for structured output can reduce malformed model responses by asking the model to return a machine-readable object such as JSON [@google_structured_outputs; @xai_structured_outputs]. In software localization, however, valid JSON is not enough: placeholders, markup, Qt mnemonic markers, and whitespace still need rule-based checks. This workflow therefore uses structured JSON output only for translated text spans. Protected tokens are not generated by the model at all; they are reinserted by Python code. In this paper, the phrase structured generation refers to the JSON text-span response, while token preservation refers to the separate rule-based layer.

Semantic quality is evaluated separately. MQM is an analytic translation quality evaluation framework applicable to human translation, machine translation, and AI-generated translation [@mqm_what_is; @lommel_mqm_2014]. @kocmi_federmann_2023 introduced GEMBA-MQM, an LLM-based method for detecting translation quality error spans using a fixed few-shot prompt and an MQM-style rubric. In this paper, LLM-as-a-judge is used as scalable assisted annotation, not as the ground truth. Model and condition labels are hidden from the judge, and rule-based structural results are reported separately from semantic MQM scores.

---

## 3. Method

### 3.1 Workflow overview and model backends

The workflow has six stages:

1. Parse the Qt `.ts` file into translation units.
2. Split each source string into protected tokens and translatable text spans.
3. Retrieve glossary hints from tabular terminology resources for the source segment.
4. Ask an interchangeable LLM backend to translate only the text spans.
5. Reinsert protected tokens by rule and select a structurally valid candidate.
6. Validate the generated `.ts` file and emit structural, rule-based QA, and MQM reports.

```{figure} workflow.svg
:name: fig-workflow-overview
:alt: Workflow for Qt .ts localization
:width: 100%

Workflow for Qt `.ts` localization. Stages 2--5 repeat for each translation unit; protected tokens are reinserted before a structurally valid candidate is selected.
```

Figure 1 separates rule-based processing from model-dependent generation. Parsing, token splitting, glossary lookup, token reinsertion, candidate selection, and validation are implemented once in a Python script and are shared across all backends. The backend adapter is the only part that changes when the workflow calls Gemini, Grok, or a local TAIDE model. This design makes the comparison focus on whether the same workflow remains reliable across different model access modes, rather than on provider-specific code.

The rule-based and glossary layers are applied uniformly across all model backends. The only backend-specific component is the adapter that converts a common request object into a cloud API call or a local inference call.

**Table 1. Model backends.**

| Backend | Access | Model ID | Role |
|---|---|---|---|
| Gemini | Cloud  | `gemini-3.1-flash-lite` | Translation backend |
| Grok | Cloud  | `grok-4.3` | Translation backend and MQM judge |
| TAIDE | Local | `taide/Gemma-3-TAIDE-12b-Chat-2602` | Local translation backend |

Model-version details are kept out of the main table to reduce width. Gemini and Grok are cloud backends with support for structured outputs [@google_gemini_flash_lite_api; @google_structured_outputs; @xai_grok_43; @xai_structured_outputs]. The TAIDE backend is a Hugging Face Hub checkpoint whose model card states that it is based on `google/gemma-3-12b-pt`; the base Gemma 3 12B pre-trained checkpoint and Gemma 3 technical report provide the relevant Gemma 3 model context [@taide_gemma3_12b_2602; @google_gemma3_12b_pt; @gemma3_technical_report]. Because cloud model aliases and model repositories are dynamic, the reproducibility artifact records API request dates and prompt versions, as well as the local checkpoint revision, download date, license terms, inference engine, quantization, GPU, VRAM, and decoding settings.

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

where $x_j$ are translatable or copyable text spans and $\tau_j$ are protected formatting-language tokens. The protected token sequence is:

$$
T(s) = (\tau_1, \tau_2, \ldots, \tau_k).
$$

For any accepted output $\hat{t}$, the structural preservation condition for token classes checked by the validator is:

$$
T_r(s) = T_r(\hat{t}) \quad \forall r \in \mathcal{R},
$$

where $\mathcal{R}$ is the set of token extractors, such as Qt placeholders, brace placeholders, printf placeholders, HTML/XML tags, entities, numeric tokens, newlines, and Qt mnemonic markers.

Suppose the source message is shown as follows, with the newline escaped as `\n` for readability:

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
        qt_mnemonic_markers: ["&A"],
        newlines: ["\n"],
        qt_placeholders: ["%1"],
        brace_placeholders: ["{feature_id}"]
       }
)
```

The splitter then produces this alternating sequence for $s$:

```text
text_0   = "Open "
token_1 = "&A"
text_1   = "Attribute Table"
token_2 = "\n"
text_2   = "Layer: "
token_3 = "%1"
text_3   = ", feature ID: "
token_4 = "{feature_id}"
```

The protected token sequence $T$ is therefore:

```text
T(s) = ("&A", "\n", "%1", "{feature_id}")
```

The model is asked to translate only the text spans and may return JSON such as:

```json
{
  "text_0": "開啟",
  "text_1": "屬性表",
  "text_2": "圖層：",
  "text_3": "，圖徵 ID："
}
```

Python then restores the protected tokens by rule. Placeholders and line breaks are restored in place. The Qt mnemonic marker is restored as a suffix at the end of the translated label:

```text
開啟屬性表(&A)
圖層：%1，圖徵 ID：{feature_id}
```

The accepted output still has the same checked token sequence:

```text
T(accepted output) = ("&A", "\n", "%1", "{feature_id}")
```

This example also shows the boundary of the model's responsibility. The model chooses the Chinese terms for `Open`, `Attribute Table`, `Layer:`, and `Feature ID:`, but it does not need to regenerate the accelerator marker, placeholder, newline, or brace placeholder.

### 3.3 Structure-preserving masked template translation

Each source string is split into translatable `TEXT` spans and protected tokens. This separation is referred to as masking in the experimental conditions. Protected tokens include:

```text
Qt placeholders:                 %1, %2, %n, %L1
Brace placeholders:              {0}, {feature_id}
printf placeholders:             %s, %d, %.2f
HTML/XML tags:                   <b>, </b>, <p align="right">
Escaped entities:                &amp;, &lt;, &gt;, &nbsp;
Escaped controls:                \n, \t, \r
Actual controls:                 newline, tab, carriage return
Qt mnemonic markers:             the ampersand plus the following key, such as &F in &File or &A in &Attribute
Literal ampersand escape in Qt:  &&
Numeric literals:                100, 3.44, -1
```

The LLM is not used to reproduce these tokens. It translates only the human-language spans and returns JSON keyed by text-span identifiers. The Python script then restores the protected tokens by rule.

Example source:

```xml
<p align="right">Algorithm author: {0}</p>
```

Template spans:

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

This design makes preservation of checked token classes rule-based for accepted candidate translations. The workflow can preserve the defined format rules, but it cannot guarantee that the translation has the right meaning, sounds natural, or works well in the user interface.

### 3.4 Glossary retrieval from tabular resources

Project glossaries are provided as spreadsheet-style terminology tables. The method only assumes that each table can be normalized into records containing:

```text
source_term, target_terms, forbidden_terms, priority, domain, note
```

Here, `forbidden_terms` are target-language expressions that the project glossary marks as discouraged or disallowed for a given source term.

To avoid scanning every glossary entry for every segment, source terms are tokenized, normalized, and indexed by n-gram keys:

$$
I[n, g] = \{e \in G : \operatorname{key}_n(e.source\_term) = g\},
$$

where $G$ is the glossary and $g$ is a normalized n-gram key. For a source segment $s$, the retrieved hint set is:

$$
H(s) = \operatorname{TopK}\left(\bigcup_{n=1}^{N_{max}} I[n, \operatorname{ngrams}_n(s)], K\right).
$$

Here, $H(s)$ denotes the glossary hints retrieved for segment $s$, $N_{max}$ is the maximum n-gram length considered, and $K$ is the maximum number of hints passed to the model. In the implementation, matched entries are sorted by source-term length so that more specific multi-word terms are preferred before shorter terms.

Glossary hints are passed to the LLM as contextual guidance, not mechanical replacements. This allows the model to adapt terminology to context while keeping prompt size controlled.

### 3.5 Candidate selection and non-accepted rows

Each segment may produce multiple candidate translations. The default production condition uses three candidates. A Python script reassembles each candidate, validates it, and selects the best valid candidate by a rule-based quality-risk score:

$$
Q(\hat{t}) = \sum_{e \in E(\hat{t})} w(severity(e)) + \lambda_1 R_{latin}(\hat{t}) + \lambda_2 R_{len}(s,\hat{t}).
$$

Here, $\hat{t}$ is a candidate translation for source segment $s$, $E(\hat{t})$ is the set of validation issues detected in the candidate, and $w(severity(e))$ maps each issue severity to a fixed penalty weight. $R_{latin}(\hat{t})$ penalizes unexpected Latin-script residue, $R_{len}(s,\hat{t})$ penalizes extreme source-target length ratios, and $\lambda_1$ and $\lambda_2$ are fixed weights for these two penalty terms.

The accepted candidate is:

$$
j^* = \arg\min_{j : E_{blocking}(\hat{t}_{u,j})=\emptyset} Q(\hat{t}_{u,j}).
$$

Here, $\hat{t}_{u,j}$ is the $j$-th candidate for translation unit $u$, and $E_{blocking}(\hat{t}_{u,j})$ is the subset of validation issues that block automatic acceptance.

If no generated candidate passes strict validation, the row is not treated as accepted localized content and is logged for review. This review state is not used as a key quality metric because it combines several operational causes: no valid candidate, backend exception, malformed JSON, or intentionally conservative rejection. Detailed review-state counts are reported in the accompanying reproducibility artifact. This paper focuses on structural failure rate, rule-based content/completion diagnostics, and MQM semantic quality.

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
Output: translated TS file, translation log, rule-based reports

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
            add (t_hat, rule-based_risk_score(t_hat)) to V

    if V is not empty:
        write the lowest-risk structurally valid candidate as accepted output
    else:
        write a review-required row and log diagnostics

validate XML well-formedness
emit CSV, JSON, and Markdown reports
```

---

## 4. Implementation and Experimental Conditions

The implementation is a Python command-line workflow. The main runner builds fixed subsets, runs the C0--C4 experimental conditions, writes generated `.ts` files, and calls evaluation and scoring scripts. The same selected subset is reused across model backends to enable paired comparisons.

**Table 2. Experimental conditions.**

| Condition | Masking | Glossary hints | Candidates | Purpose |
|---|---:|---:|---:|---|
| C0 (Direct) | no | no | 1 | Direct translation without masking |
| C1 (Complete workflow) | yes | yes | 3 | Production condition |
| C2 (Direct + glossary) | no | yes | 3 | Tests glossary/candidates without masking |
| C3 (Masked without glossary) | yes | no | 3 | Tests masking without glossary hints |
| C4 (Single candidate)  | yes | yes | 1 | Tests the value of generating multiple candidates |

C0--C4 are experimental conditions for the same workflow. Evaluation on the complete corpus focuses on C1 because it is the pre-specified production workflow. Scaling all experimental conditions to the complete corpus would primarily increase cost and review burden rather than change the deployment question: whether the selected production workflow remains structurally safe at QGIS scale.

### 4.1 Direct generation and residual risk

A natural question is whether stronger prompting is enough. C0 and C2 should be interpreted conservatively: they demonstrate residual risk under the implemented direct prompts, but they are not an exhaustive benchmark of every possible prompt-only or constrained-decoding design. In these conditions without masking, the model sees the full source string and remains responsible for reproducing protected formatting-language tokens. C2 additionally receives glossary hints and three candidates. These settings can improve semantic quality, but they still leave protected-token preservation inside the model generation task. Masked conditions change the problem decomposition: the LLM translates human-language spans, while rule-based code preserves protected tokens.

### 4.2 Validation fixtures and smoke tests

The artifact includes a bundled mini evaluation fixture for exercising the rule-based layer. The fixture contains archived 100-segment C0--C4 .ts outputs, condition metadata, selected-segment metadata, and evaluation outputs. The default no-key reproduction reruns the rule-based evaluator and scoring scripts on these archived .ts files, so that token checks, accelerator handling, XML well-formedness checks, condition comparison, scoring behavior, MQM request planning, and table generation can be verified without calling any model API.

The fixture is not presented as a full regeneration of the paper experiments: it does not rerun LLM translation, candidate generation, glossary-assisted prompting, or token reassembly during generation. Those steps are covered by the workflow scripts and can be exercised through optional cloud/local reruns, while the default no-key path provides a stable offline check of the rule-based evaluation and reporting code path.

---

## 5. Evaluation

This section evaluates the effectiveness of our workflow. The purpose of the evaluation is not to rank the models we use. Rather, it is to validate the method we propose. It has two reported layers and one complementary semantic-review protocol.

1. **Condition comparison:** Grok, Gemini, and TAIDE are evaluated under C0--C4 on the same 3,000-segment stratified subset of source messages.
2. **Complete corpus:** Grok, Gemini, and TAIDE are evaluated under C1 on all 28,924 QGIS source messages.
3. **Semantic review:** MQM-style sampled judging evaluates semantic adequacy, terminology, fluency, and locale style on the C0--C4 outputs, while remaining separate from rule-based structure scoring.

The subset is stratified by source-string features that are likely to affect localization reliability, including plural/numerus messages, Qt placeholders, accelerator markers, HTML/XML content, glossary hits, numeric or code-like content, newline/control characters, long strings, and ordinary strings. Because these categories can overlap, a segment may satisfy more than one stratum; the final subset is fixed and reused across all model backends and experimental conditions.

### 5.1 Structural metrics

For condition $c$ and $N$ checked segments, the structural failure rate is:

$$
\hat{p}_c = \frac{1}{N}\sum_{u=1}^N \mathbb{1}\left[E_{struct}(u,c) \neq \emptyset\right].
$$

For zero observed failures, the one-sided approximate 95% upper bound is reported using the rule of three [@hanley_lippmanhand_1983]:

$$
p_{upper} \approx \frac{3}{N}.
$$

For the 3,000-segment subset, this corresponds to approximately 0.1000%. For the 28,924-message complete corpus, it corresponds to approximately 0.0104%.

### 5.2 Form/Content Completeness QA Metrics

For **form/content completeness QA**, we mean quality assessment about untranslated or partially translated text, empty outputs, missing translation elements, rows with no valid candidates, glossary target missing counts, and forbidden term counts. These metrics are engineering diagnostics, not a complete translation-quality score. A condition can improve structural safety while lowering rule-based content/completion QA if it creates more review-required or potentially untranslated rows.

### 5.3 MQM-style semantic review metrics

Semantic quality is evaluated with MQM-style scoring. For segment $u$, let $P_u$ be the set of MQM penalties assigned by the judge. The weighted MQM error rate is:

$$
MQM\text{-}ER = 1000 \times \frac{\sum_{u=1}^N\sum_{p \in P_u} p}{\sum_{u=1}^N |s_u|}.
$$

Lower values indicate fewer weighted semantic errors per 1,000 source characters. The auxiliary MQM score on a 0--100 scale is reported for readability; the error rate is the main semantic metric.

The MQM analysis is not framed as a claim that masking should improve semantic quality. The trade-off question is whether the semantic cost of masking is small enough to justify the structural safety gain. Let:

$$
\Delta_{MQM} = MQM\text{-}ER(C1_{masked}) - MQM\text{-}ER(C0_{direct}).
$$

A value near zero indicates no material semantic penalty; a small positive value indicates a modest semantic cost; a negative value indicates that the masked workflow also improves semantic quality. In all cases, MQM is interpreted separately from structural failure rate.

---

## 6. Results

### 6.1 Condition-comparison subset: structural safety and semantic trade-off

The full C0--C4 condition outputs are archived in the accompanying reproducibility artifact. Table 3 reports the four conditions central to the main trade-off comparison: C0 direct baseline, C1 complete workflow, C2 direct generation with glossary hints, and C4 single-candidate masked workflow. C3 is omitted from the compact table because it mainly isolates the glossary component; it is included in the artifact and shows the same zero-observed-structure-failure pattern as the other masked conditions.

**Table 3. Compact condition-comparison summary on the 3,000-segment subset.** `Rule QA` abbreviates rule-based content/completion QA, an engineering diagnostic rather than a complete translation-quality score.

| Backend | Cond. | Short name | Structure failure % ↓ | Rule QA ↑ | MQM-ER ↓ |
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

The main observation is structural rather than semantic. Under the implemented conditions without masking, C0 and C2 retain non-zero structural risk for every backend: Grok C0 and C2 have structure-failure rates of 5.67% and 5.13%, Gemini C0 and C2 have rates of 8.70% and 9.30%, and TAIDE C0 and C2 have rates of 30.10% and 25.17%. These results do not claim to rule out all possible prompt-only methods; they show that direct generation leaves a residual token-preservation risk in this workflow. In contrast, all masked conditions produce zero observed structure failures under the checked token classes. The full item-level breakdown shows that direct generation failures concentrate in newline preservation, Qt shortcut-marker preservation, HTML/XML tag preservation, and placeholder preservation. Detailed counts are reported in the accompanying reproducibility artifact to keep the main text concise.

MQM shows the expected semantic trade-off. C2, the direct condition with glossary hints, obtains the lowest MQM error rate for all three backends, but it also retains non-zero structural failure rates. C1 has a higher MQM error rate than C0 and C2, especially for cloud models, but removes the observed structural failures under the implemented extractors. This supports the paper's main distinction: direct generation can be more fluent or semantically preferred, while masked settings are safer for deployment.

### 6.2 Effect of generating multiple candidates

C1 and C4 isolate the value of generating multiple candidates. Both use masking and glossary hints from tabular resources; C1 uses three candidates and C4 uses one. Both achieve zero observed structure failures, so candidate count is not the source of structural safety. Instead, multiple candidates mainly improve robustness by providing more candidate translations to choose from.

For cloud backends, the effect is small. Relative to C4, C1 improves rule-based content/completion QA by +0.17 for Grok and +0.15 for Gemini. For TAIDE, the effect is larger: rule-based content/completion QA improves by +3.80, and the average valid candidate count increases from 0.689 to 2.119. MQM also favors C1 over C4 for all three backends, but the differences are modest relative to repeated-judge variability. We therefore interpret three-candidate selection as a robustness mechanism, not as the primary safety contribution. The primary safety mechanism remains masking plus rule-based reassembly.

### 6.3 Complete-corpus C1 production results

Complete-corpus C1 experiments were run on all 28,924 QGIS source messages for each backend. C1 is the production workflow: masking, glossary hints from tabular resources, and three candidates.

**Table 4. C1 production-condition comparison on the complete corpus.**

| Backend | Messages checked | Structure failure % ↓ | Structure score ↑ | Rule QA ↑ | Avg. valid candidates | Possibly untranslated |
|---|---:|---:|---:|---:|---:|---:|
| Grok 4.3 | 28,924 | 0.00 | 100.000 | 92.11 | 2.938 | 1,394 (4.82%) |
| Gemini 3.1 Flash-Lite | 28,924 | 0.00 | 100.000 | 90.09 | 2.935 | 1,825 (6.31%) |
| TAIDE 12B | 28,924 | 0.00 | 100.000 | 65.13 | 2.052 | 6,674 (23.07%) |

The complete corpus result confirms that the subset finding scales to the complete QGIS source file: all three backends produce zero observed structural failures under the checked token classes in C1. The local TAIDE backend is structurally safe under the workflow, but its rule-based content/completion QA is much lower. This indicates that the local backend is a viable offline deployment path for structural preservation, while currently requiring substantially more review than the cloud backends.

---

## 7. Reproducibility and Artifact Availability

The accompanying repository is available at <https://github.com/wendy062644/qgis-llm-localization-workflow>. A versioned release tag will be archived for the final proceedings version. The artifact is organized around three reproducibility levels so that reviewers can distinguish no-key rule-based checks from costlier model reruns.

**Table 5. Reproducibility levels.**

| Level | Command or location | New model calls? | What it verifies |
|---|---|---:|---|
| Mini rule-based evaluation | `python scripts/run_repro.py full-mini`; `experiments/demo_ablation_grok_100/` | No | Reruns rule-based evaluation, scoring, condition comparison, and MQM request generation on bundled 100-segment C0--C4 `.ts` outputs. |
| Extended archived-output scoring | `python scripts/run_repro.py score --experiment <experiment> --force-eval` | No | Uses a downloaded paper-experiment directory to recompute structure and rule-based content/completion metrics from archived generated `.ts` files rather than from preformatted table rows. |
| Optional MQM or translation rerun | `python scripts/run_repro.py mqm ... --run-grok`; `python scripts/run_repro.py translate ...` | Yes | Rebuilds MQM judge outputs or regenerates a small translation smoke run using API credentials or local inference. |

The default mini rule-based evaluation is reviewer-friendly and offline. It uses the same C0--C4 condition definitions and the same rule-based evaluator/scorer as the paper experiments, but on a smaller 100-segment archived-output fixture so that it can run quickly. It should be interpreted as an executable validation of the rule-based evaluation, scoring, comparison, and request-planning code paths, not as a full regeneration of LLM translation outputs or as a substitute for the 3,000-segment and 28,924-message experimental artifacts.

The compact release contains the mini fixture, source `.ts` file, tabular glossary resources used in the experiments, C0--C4 configuration files, workflow scripts, archived demo `.ts` outputs, workflow manifests, rule-based evaluation outputs, MQM request-planning outputs, and an `.env.example` file. The final proceedings artifact includes paper-facing table artifacts for the reported 3,000-segment C0--C4 condition comparison, complete-corpus C1 comparison, and MQM summaries. Full paper-scale archived-output recomputation is provided through a larger extended experiment archive containing the generated `.ts` outputs for the reported 3,000-segment C0--C4 condition comparison and 28,924-message C1 runs. With that archive, the same scoring command recomputes rule-based metrics from archived outputs without new model calls. Rerunning translation or MQM judging requires the appropriate API credentials or local inference hardware. API keys are excluded from the repository and are supplied through environment variables such as `XAI_API_KEY` and `GEMINI_API_KEY`.

In addition to the reproducibility artifact, the released Traditional Chinese localization files are deposited as public datasets in depositar with persistent ARK identifiers. The released datasets include the QGIS 4.0 zh-Hant Translation Dataset (TS/QM Files), available at <https://pid.depositar.io/ark%3A37281/k5f167n46> with ARK `ark:37281/k5f167n46`, and the Long Term Release QGIS 3.44 zh-Hant Translation Dataset (TS/QM Files), available at <https://pid.depositar.io/ark%3A37281/k5g11462n> with ARK `ark:37281/k5g11462n`. These deposits provide persistent access to the released translation files, while the repository artifact described above contains the scripts, configurations, archived outputs, and evaluation reports needed to reproduce the paper's analyses.

The repository README documents the quickstart, rule-based scoring, MQM request planning, optional MQM judge run, and optional API translation smoke run.

---

## 8. Discussion and Limitations

The workflow improves reliability by separating structural safety, terminology guidance, semantic evaluation, and human review. The rule-based layer is the most objective because it checks software artifacts that should be preserved. The structure-preserving template layer strengthens this idea by removing protected tokens from the LLM generation space. Generating multiple candidates is a secondary robustness layer; it improves candidate availability, especially for the local backend, but it is not the mechanism that creates the structural guarantee.

The role of masking should be interpreted carefully. Masking does not solve semantic adequacy by itself, and the MQM results show a modest semantic cost for C1 compared with C0 and C2, the two conditions without masking. However, this is a different risk category from interface breakage. C0 and C2 provide evidence of residual risk under the implemented direct generation settings, not an exhaustive dismissal of all prompt-only approaches. C2 can be semantically stronger, but it still leaves non-zero structural failure rates in this experiment. In practical terms, masking trades a measured semantic cost for a much stronger guarantee that the localized interface remains structurally usable under the checked token classes.

The local LLM results should be read as evidence for deployability, not model superiority. TAIDE achieves the same zero observed structural failure result under C1, but its rule-based content/completion QA and review-workload indicators are much worse than the cloud backends. In the archived status diagnostics, 7,265 of 28,924 TAIDE C1 rows, or 25.12%, entered a review-required source-fallback state. This is not a fatal flaw for the workflow, because the workflow prevents those rows from being silently accepted as structurally valid localized content. It does mean that the local backend currently requires substantially more human review before release; in the present configuration, TAIDE should be treated as an offline deployment path with a larger review queue rather than as a drop-in substitute for the cloud backends.

The glossary layer depends on glossary quality. Because glossary matches are hints rather than replacements, the workflow avoids many unnatural literal substitutions, but terminology correctness still requires evaluation.

Semantic review is necessary because the rule-based layer does not evaluate adequacy, fluency, domain correctness, or target-locale style. LLM judges can be useful for scalable review, but they are assisted annotation rather than ground truth. Human review remains necessary for release-critical strings, ambiguous GIS terms, and user-facing messages with high impact.

### 8.1 Practical deployment recommendation

For a FOSS localization team, the recommended deployment workflow is:

1. Run the selected production condition.
2. Accept only rows with zero blocking structural issues.
3. Treat review-required rows as incomplete, not as translated content.
4. Prioritize review by English residue, terminology warnings, semantic severity, and review status.
5. Run XML well-formedness and rule-based validation before each release candidate.
6. Use human review for final acceptance of UI-critical strings and domain terminology.

The workflow preserves checked token classes and creates auditable review queues; it does not eliminate all localization errors or prove translation quality.

---

## 9. Conclusion

This paper presents a Python workflow for reliable localization of QGIS Qt `.ts` files with LLM support, without tying the method to a single model backend. The workflow supports both cloud-based and local LLMs, uses tabular glossary resources as contextual terminology hints, and applies structure-preserving template translation to preserve protected tokens through rule-based reassembly.

Condition-comparison results on the 3,000-segment subset show that conditions without masking produce non-zero structural failure rates for Grok, Gemini, and TAIDE, while masked template conditions produce zero observed structure failures under checked token classes for all three backends. Complete-corpus C1 results extend this finding to all 28,924 QGIS source messages: Grok, Gemini, and TAIDE all have zero observed structure failures under the implemented extractors. Semantic results using MQM show that masking is not a semantic optimizer: C1 has a modestly higher MQM error rate than the direct baselines, and C2 has the best semantic scores but remains structurally unsafe. C1 versus C4 further shows that generating three candidates helps robustness, especially for TAIDE, but the primary safety mechanism remains masking plus rule-based reassembly.

The broader contribution is methodological. Reliable FOSS localization should not depend on raw LLM output alone. It should be built as a reproducible workflow in which rule-based validation protects software structure, glossary retrieval supports terminology consistency, MQM evaluation with LLM assistance scales semantic review, and human reviewers provide final credibility.

---

## Acknowledgements and AI Disclosure

Generative AI was used for English wording and formatting assistance. The authors verified the technical claims, references, experimental results, source code, and final manuscript text. LLM outputs used for translation or MQM-style judging are treated as assisted annotations, not the ground truth.

## Funding

This research is supported in part by an Academia Sinica Thematic Project grant (no. AS-TP-114-L01; _Sound Atlas: Mapping the Changing Terrestrial, Marine, and Cultural Soundscapes for a Sustainable Island Social-Ecological System_) and a grant from the National Science and Technology Council of Taiwan (no. NSTC 114-2621-M-001-001; _Advancing Research Data Infrastructures and Management Practices: Tools, Services, and Communities_).


---

## References

```{bibliography}
```
