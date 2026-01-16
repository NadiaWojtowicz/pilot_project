# Agent-Based Model of Political Opinion Dynamics in Danish Media

This repository contains data processing scripts, network analyses, and annotation workflows for an agent-based modeling study examining political opinion dynamics in Aarhus news coverage.

---

## 1. Scripts

R Markdown scripts for data preprocessing, topic classification, translation, human annotation processing, and network generation.

### Topic Assignment & Data Initialization

**`new_data_labels.Rmd`**
- Initializes binary topic indicators (topic_1 through topic_10) for full article corpus
- Maps 8 topic-specific CSV files to their corresponding topic numbers (topics 5 and 6 have no data files)
- Topic mapping: 1=privatization, 2=budget savings, 3=municipal tax, 4=business costs, 7=parking spaces, 8=harbor expansion, 9=traffic limitations, 10=stadium Kongelunden
- Matches articles by `article_id` from topic files to `file` column in main dataset
- **Input**: `/work/pilot/data/clean/clean_articles.csv`, 8 topic files from `/work/pilot/data/clean/topics/`
- **Output**: `/work/pilot/data/clean/all_articles_with_topics.csv`

**`clean_splitting_articles.Rmd`**
- Merges 4 batch LLM classification result files (0-330, 331-661, 662-992, 993-1322)
- Removes existing `id` columns from individual batches and creates new sequential ID
- Filters to keep only articles with at least one topic hit (any topic_1...topic_10 == 1)
- Generates distribution statistics showing number of topics per article
- Splits filtered data into 10 topic-specific CSV files (articles with multiple topics appear in multiple files)
- **Input**: `/work/pilot/data/processed/0_330_results.csv`, `331_661_results.csv`, `662_992_results.csv`, `993_1322_results.csv`
- **Output**: `/work/pilot/data/processed/FINAL_1323_results.csv`, `/work/pilot/data/processed/FINAL_hits_only.csv`, 10 files in `/work/pilot/data/processed/topic_splits/` (columns: id, url, text_excerpt, topic_1...topic_10, model_output)

**`split_and_hits_datanew.Rmd`**
- Filters articles from full corpus with topic assignments to keep only those with topic hits
- Creates 10 topic-specific CSV files in `/work/pilot/data/processed/split_topics/`
- **Input**: `/work/pilot/data/clean/all_articles_with_topics.csv`
- **Output**: `/work/pilot/data/processed/articles_with_hits.csv`, 10 split files

**`merging_all_articles.Rmd`**
- Adds sequential ID column to complete topic matrix
- Analyzes topic distribution across articles using `/work/pilot/data/processed/full_df.csv`
- Splits data into 10 topic-specific files in `/work/pilot/data/processed/all_topics_split/`
- **Input**: `/work/pilot/data/clean/all_topics_matrix.csv`, `/work/pilot/data/processed/full_df.csv`
- **Output**: `/work/pilot/data/processed/all_topics_results.csv`, 10 files in `/work/pilot/data/processed/all_topics_split/` (columns: id, url, text_excerpt, topic_1...topic_10, model_output)

### Translation

**`google_translation.Rmd`**
- Translates Danish article text to English using Google Cloud Translation API v2
- Implements row-by-row translation with error handling and progress saving
- Skips already-translated rows (checks for existing `eng_text` values)
- Adds `eng_text` column to dataset
- Includes 0.1 second delay between API calls for rate limiting
- **Dependencies**: httr, jsonlite, readr, dplyr
- **Input**: `/work/pilot/data/processed/full_df.csv` (Danish text in `text` column)
- **Output**: Updated `/work/pilot/data/processed/full_df.csv` with `eng_text` column

### Annotation Preparation

**`preprocessing_hits_only.Rmd`**
- Adds sequential ID to full article corpus, saves as `/work/pilot/data/clean/all_articles_IDs.csv`
- Drops `text_excerpt` column from processed topics, joins full article `text` from ID'd corpus
- Filters to keep only articles with topic hits
- Generates topic distribution statistics
- Splits into 10 topic-specific files for annotation
- **Input**: `/work/pilot/data/clean/clean_articles.csv`, `/work/pilot/data/processed/all_topics_split.csv`
- **Output**: `/work/pilot/data/clean/all_articles_IDs.csv`, `/work/pilot/data/processed/hits_only_data.csv`, 10 files in `/work/pilot/data/processed/topic_splits/` (columns: id, url, text, topic_1...topic_10, model_output)

**`clean_for_label.Rmd`**
- Selects columns needed for Label Studio annotation interface
- Keeps: file, text, topic_1 through topic_10, eng_text
- **Input**: `/work/pilot/data/processed/articles_with_hits.csv`
- **Output**: `/work/pilot/data/processed/articles_for_label.csv`

### Human Annotation Processing

**`processing_humanmodel.Rmd`**
- Parses Label Studio JSON export (`human_lable.json`) to extract human topic annotations
- Extracts single-choice labels from annotation results (values: "1"-"10" or "none")
- Creates binary prediction columns `pred_topic_1` through `pred_topic_10` based on annotations
- Removes duplicate annotations by `file` (keeps first occurrence)
- Merges with LLM predictions from `/work/pilot/data/processed/label2.csv` (renames `url` to `file`)
- Filters out all articles labeled as "none" (none == 1)
- Generates topic distribution statistics and histogram visualization
- Retains only essential columns: file, text, eng_text, pred_topic_1...pred_topic_10
- **Input**: `/work/pilot/data/processed/human_lable.json`, `/work/pilot/data/processed/label2.csv`
- **Output**: `/work/pilot/data/processed/cleaned_data_no_none.csv`, `/work/pilot/data/processed/human_labels_histogram.png`

### Network Generation & Visualization

**`network_visual-1.Rmd`**
- Parses Label Studio sentiment annotation export (`finished_label.json`) with 30-class labels (10 topics × 3 sentiments)
- Decodes numeric labels (1-30) into topic-sentiment-opinion tuples using modular arithmetic
- Encoding scheme: sentiment = ((label-1) mod 3) + 1, topic = ceiling(label/3), opinion ∈ {-1, 0, 1}
- Creates nodes (topic-sentiment pairs) and weighted edges (co-occurrence in same article)
- Filters to articles with 2+ distinct topic-sentiment nodes for edge construction
- Generates 6 network visualizations using igraph and ggraph:
  1. **Force-directed network** (Fruchterman-Reingold layout): Main co-occurrence structure
  2. **Circular layout**: Symmetric arc-based view
  3. **Community detection** (Louvain algorithm): Identifies media framing clusters
  4. **Symmetric heatmap**: Full co-occurrence matrix (includes diagonal and reverse edges)
  5. **Publication-ready network** (stress layout, edges filtered at weight_norm ≥ 0.1)
  6. **Topic-level 10×10 heatmap**: Aggregated co-occurrence by topic number only
- Calculates network metrics: node count, edge count, density, mean/max edge weight
- **Dependencies**: jsonlite, igraph, ggraph, tidygraph, tidyverse, scales
- **Input**: `/work/pilot/data/processed/finished_label.json`
- **Output**: 
  - `/work/pilot/data/processed/networks/topic_sentiment_nodes.csv`
  - `/work/pilot/data/processed/networks/topic_sentiment_edges.csv`
  - `/work/pilot/data/processed/networks/articles_with_nodes.csv`
  - 6 PNG network visualizations in `/work/pilot/data/processed/networks/`

---

## 2. ABM Initialization Data

Data files containing conditional probability matrices for agent initialization in the agent-based model. These files encode opinion correlations between topics based on empirical data.

### Kandidattest Agent Initialization

**`kandidattest_abm_init_complete.csv`**
- Conditional probability matrix for initializing political candidate agents
- Each row represents: given topic X with sentiment Y, probability of holding sentiment Z on topic W
- **Columns**:
  - `topic_given`: Topic number (1-10) that agent currently holds opinion on
  - `sentiment_given`: Sentiment on given topic ("negative", "neutral", "positive")
  - `marginal_probability`: Proportion of kandidattest responses with this topic-sentiment combination
  - `topic_predicted`: Target topic number for conditional probability
  - `sentiment_predicted`: Target sentiment for conditional probability
  - `conditional_probability`: P(sentiment_predicted on topic_predicted | sentiment_given on topic_given)
  - `correlation`: Pearson correlation between opinions on topic_given and topic_predicted
  - `n_observations`: Number of kandidattest responses used to calculate probabilities
- Used to initialize agents with realistic opinion correlation structures
- Note: All `sentiment_given="neutral"` rows have 0 probability (neutral not observed in kandidattest data)
- **Format**: 270 rows per sentiment combination (10 topics × 3 sentiments × 9 other topics)

**`kandidattest_abm_init_complete.json`**
- JSON format of `kandidattest_abm_init_complete.csv` (same data, different format)

### Article-Based Agent Initialization

**`articles_abm_init_complete.csv`**
- Conditional probability matrix for initializing media/article agents
- Structure identical to kandidattest file but based on co-occurrence patterns in labeled news articles
- Derived from human-annotated article sentiment labels
- Sparse data: many combinations have `n_observations=1` or small sample sizes
- **Columns**: Same as kandidattest file (topic_given, sentiment_given, marginal_probability, topic_predicted, sentiment_predicted, conditional_probability, correlation, n_observations)
- Used for initializing agents representing media discourse patterns

**`articles_abm_init_complete.json`**
- JSON format of `articles_abm_init_complete.csv` (same data, different format)

**`articles_abm_init_complete_smoothed.csv`**
- Smoothed version of article initialization probabilities to handle sparse data
- Applies Laplace/additive smoothing to conditional probabilities to avoid zero-probability events
- **Additional columns**:
  - `conditional_probability_smoothed`: Smoothed probability values replacing zeros with small non-zero values
  - `smoothed`: Boolean flag (TRUE) indicating which rows were smoothed
- Recommended for ABM initialization to ensure all opinion transitions have non-zero probability
- Prevents edge cases where agents cannot adopt certain opinion combinations

---

## 3. Network Data Files

### Articles Network (Media Discourse Patterns)

Network files capturing topic-sentiment co-occurrence patterns in human-annotated news articles.

**`topic_sentiment_nodes-1.csv`**
- Node table for articles network (15 unique topic-sentiment combinations observed in labeled articles)
- **Columns**:
  - `node`: Node identifier in format `topic_X_SENTIMENT` (e.g., "topic_1_neutral", "topic_7_negative")
  - `topic`: Topic number (1-10)
  - `sentiment`: Sentiment label ("negative", "neutral", "positive")
  - `opinion`: Numeric opinion value (-1 = negative, 0 = neutral, 1 = positive)
- Represents all observed topic-sentiment pairs in annotated article dataset
- Note: Not all 30 possible combinations (10 topics × 3 sentiments) appear due to sparse data

**`topic_sentiment_edges-1.csv`**
- Edge table for articles network (18 unique co-occurrence relationships)
- **Columns**:
  - `from`: Source node identifier (format: `topic_X_SENTIMENT`)
  - `to`: Target node identifier (format: `topic_X_SENTIMENT`)
  - `weight`: Raw co-occurrence count (number of articles mentioning both topic-sentiment pairs)
  - `weight_norm`: Normalized weight (0-1 scale, divided by maximum weight)
- Edges represent articles that discuss multiple topic-sentiment pairs
- Strongest edges: `topic_10_neutral` ↔ `topic_8_neutral` (weight=2), `topic_7_negative` ↔ `topic_9_negative` (weight=2)

**`articles_with_nodes.csv`**
- Long-format article-level data showing which articles contain which topic-sentiment nodes
- **Columns**:
  - `file`: Article filename/identifier
  - `topic`: Topic number mentioned in article
  - `sentiment`: Sentiment expressed on that topic
  - `opinion`: Numeric opinion value
  - `node`: Node identifier
- Articles with multiple topic-sentiment annotations appear on multiple rows
- Total: 51 rows from 23 unique articles (some articles discuss multiple topic-sentiment pairs)

**Network Visualizations (JPG files)**
- `articles_network_force.jpg`: Force-directed layout (Fruchterman-Reingold algorithm)
- `articles_network_circular.jpg`: Circular layout emphasizing network symmetry
- `articles_network_communities.jpg`: Louvain community detection showing media framing clusters (6 structural clusters identified)
- `articles_network_heatmap.jpg`: Symmetric co-occurrence matrix (sparse - white cells = never co-occur)
- `articles_network_pretty.jpg`: Publication-ready network (stress layout, filtered edges with weight_norm ≥ 0.1)
- `articles_topic_heatmap.jpg`: 10×10 topic-level co-occurrence matrix (aggregated across sentiments)

### Kandidattest Network (Political Candidate Opinion Structure)

Network files capturing topic-sentiment co-occurrence patterns across political candidates from Aarhus kandidattest data.

**`kandidat_topic_sentiment_nodes.csv`**
- Node table for kandidattest network (20 nodes - all 10 topics with both negative and positive sentiment)
- **Columns**: Same structure as articles nodes (node, topic, sentiment, opinion)
- Complete coverage: All possible topic-sentiment combinations present (no neutral sentiment in kandidattest)
- Represents ideological positions taken by political candidates

**`kandidat_topic_sentiment_edges.csv`**
- Edge table for kandidattest network (171 weighted co-occurrence relationships)
- **Columns**: Same structure as articles edges (from, to, weight, weight_norm)
- Edges represent candidates holding both opinion pairs simultaneously
- Much denser than articles network (more data, 155 candidates vs ~20 articles)
- Strongest edge: `topic_8_positive` ↔ `topic_9_positive` (weight=83, weight_norm=1.00)
- Reveals ideological alignment patterns across topics

**`kandidat_with_nodes-1.csv`**
- Long-format candidate-level data (155 candidates × 10 topics = 1550 rows)
- **Columns**: Same structure as articles nodes file (file, topic, sentiment, opinion, node)
- Each candidate has exactly 10 rows (one opinion per topic)
- File identifier format: `kandidat_1`, `kandidat_2`, ..., `kandidat_155`
- Used to construct kandidattest network edges and initialize ABM candidate agents

**Network Visualizations (JPG files)**
- `kandidat_network_force.jpg`: Force-directed network of candidate opinion structure
- `kandidat_network_circular.jpg`: Circular layout showing all 20 topic-sentiment nodes
- `kandidat_network_communities.jpg`: Community detection (2 main ideological clusters identified)
- `kandidat_network_heatmap.jpg`: Symmetric 20×20 co-occurrence matrix with normalized alignment strength
- `kandidat_network_pretty.jpg`: Publication-ready candidate network visualization

---

## 4. Kandidattest Analysis & Question Selection

### Complete Question List (26 Total)

The Aarhus kandidattest includes 26 questions spanning multiple policy domains:

**Economy (7 questions)**
1. More tasks in the public sector must in future be solved by private companies
5. It is possible to save money in the public sector without affecting public welfare
8. Aarhus Municipality must make it cheaper to run a business
13. Aarhus Municipality must ensure usable shelters, even if this may mean less money for other areas
20. Aarhus Municipality must do more to attract foreign tourists
23. The city council's temporary stop for the expansion of Aarhus Harbor must be made permanent
25. The municipality must continue with the plans for the new football stadium in Kongelunden, even if the costs rise again

**Social & Welfare (5 questions)**
2. Public institutions pay too many heed to religious minorities, for example by providing meals without pork
6. Municipal tax in Aarhus Municipality must be raised and the money must be spent on better welfare
14. The elderly must be able to purchase additional services at the municipal nursing homes
15. Aarhus Municipality must use voluntary work in nursing homes to relieve staff
16. Aarhus Municipality must do more for the most vulnerable, even if this may mean less money for other areas

**Traffic & Transportation (2 questions)**
3. Investment in the road network is more urgent than investment in public transport
22. More parking spaces must be established in Aarhus Municipality
24. Car traffic in Aarhus city center must be limited, for example through one-way directions, speed reductions and a zero-emission zone

**Climate, Environment & Energy (4 questions)**
4. Canteens in public institutions must introduce two meat-free days a week
9. Several facilities for renewable energy such as wind turbines and solar cells are to be set up in Aarhus Municipality
12. It is primarily the citizens' own responsibility to protect themselves against floods and other weather events, so Aarhus Municipality must use fewer resources in the area
26. Wind turbines must be set up at the industrial port in Aarhus

**Children & Young People (3 questions)**
7. Aarhus Municipality must maintain the current number of schools, even if it is more expensive than gathering the students in larger schools
10. Aarhus Municipality must ensure a lunch arrangement in primary schools, even if it can take money from other tasks
17. Aarhus Municipality must prioritize that school pupils are mixed according to ethnicity and social background, even if it may go beyond the parents' free choice of school

**Culture, Sports & Leisure (3 questions)**
11. Aarhus Municipality must ensure better sports facilities, even if this may mean less money for other tasks
18. Higher user fees must be introduced for cultural and leisure facilities such as theatres, youth clubs and library events
21. The politicians must prevent the construction of mosques

**Housing (1 question)**
19. Too many high-rise buildings are being built

### Top 10 Discriminating Questions

Feature importance analysis using cross-validated bootstrap (50% weight), permutation importance (40% weight), and SHAP values (10% weight) identified the 10 questions with highest discriminatory power for party identification:

1. **Q24** - Car traffic limitations in city center
2. **Q17** - School mixing by ethnicity/social background
3. **Q6** - Municipal tax increase for welfare
4. **Q25** - Football stadium in Kongelunden
5. **Q1** - Privatization of public sector tasks
6. **Q21** - Prevention of mosque construction
7. **Q8** - Making it cheaper to run a business
8. **Q22** - More parking spaces
9. **Q5** - Saving money without affecting welfare
10. **Q23** - Permanent harbor expansion stop

### Interpretation of Top 10 Questions

These 10 questions capture the primary ideological cleavages in Aarhus municipal politics. The selection reflects:

**Dominant ideological dimension**: Questions Q24, Q22, Q1, Q8, and Q5 align strongly with the first principal component, representing the traditional left-right economic axis (public vs. private sector, taxation, business regulation).

**Cultural/identity dimension**: Questions Q17 and Q21 load on a secondary dimension capturing cultural liberalism vs. conservatism, particularly regarding immigration, integration, and religious minorities.

**Local development priorities**: Questions Q25 and Q23 reflect distinctive Aarhus-specific policy debates about urban development and infrastructure investment that do not align cleanly with national party lines.

**Multi-dimensional policy space**: The fact that no single dimension explains candidate variance demonstrates that Aarhus politics involves multiple independent axes of disagreement, not just left-right positioning.

These questions achieve optimal party discrimination because they force candidates to take clear positions on issues where parties have established, distinct stances. Questions with high consensus across parties (e.g., supporting schools, Q7) or ambiguous framing provide less information for distinguishing political affiliations.

### Rationale for Article Topic Selection

The 10 topics used for article scraping and sentiment annotation were selected based on the top 10 discriminating questions from kandidattest analysis. This selection strategy has important methodological implications:

**Alignment with political discourse**: By selecting topics that best distinguish between parties, we capture the issues that actually structure political debate in Aarhus, rather than imposing an arbitrary topic framework.

**Coverage limitations**: This approach prioritizes discriminatory power over comprehensive coverage. Topics with high consensus (Q7: maintaining schools, Q16: supporting vulnerable populations) or low salience in news media may be underrepresented. The selection reflects which issues *divide* political actors, not which issues are *most important* in absolute terms.

**Media-politics mapping**: The correspondence between kandidattest questions and news article topics is imperfect. Some highly discriminating questions (Q17: school mixing by ethnicity) appear rarely in news coverage, limiting our ability to construct representative media discourse networks for all dimensions. Conversely, some frequently covered topics may not appear in kandidattest.

**Data availability constraints**: Topics 5 (ethnicity/social background mixing) and 6 (mosque prevention) lack corresponding article data files (`new_data_labels.Rmd` shows no matching articles), indicating either insufficient news coverage or failed keyword matching. This gap highlights that discriminatory survey questions do not always correspond to prevalent media frames.

**Validity consideration**: The selected topics model the *discriminatory structure* of political opinions well—they capture dimensions along which candidates and parties differ. They do not necessarily model the *complete opinion space* or the *frequency distribution* of political discourse in practice. For ABM initialization, this means agent opinion structures will reflect party-differentiating issues rather than the full breadth of municipal political discussion.

### Analysis Documentation

Full statistical analysis including PCA, clustering, feature importance ranking, and network analysis is documented in `kandidat_analysis.pdf`. The analysis proceeds in several steps:

**Data Cleaning and Preprocessing**
- Raw kandidattest data cleaned and filtered to parties with ≥13 candidates (155 candidates across 6 major parties retained)
- All 26 question items rescaled to numeric values on common scale (-2 to +2) for multivariate analysis
- Missing values handled through complete-case analysis

**Reliability Analysis**
- Cronbach's alpha computed with automatic key checking to assess internal consistency
- Items with negative item-total correlations programmatically reversed
- Alpha recalculated to evaluate whether battery measures coherent ideological construct

**Principal Component Analysis (PCA)**
- PCA performed on standardized response matrix to identify main latent dimensions
- Eigenvalues and variance explained computed for each component
- PC1 interpreted as economic left-right axis, PC2 as cultural liberal-authoritarian dimension
- Loadings inspected to determine which questions define each ideological axis

**Candidate Positioning**
- Candidate positions projected into PCA space
- Party centroids calculated as mean PC scores per party
- Within-party dispersion measured as Euclidean distance from each candidate to their party centroid
- ANOVA with Tukey HSD post-hoc tests performed to assess between-party differences on PC1
- Levene's test conducted to check homogeneity of variance assumption

**Clustering Analysis**
- K-means clustering performed on first 6 principal components
- Optimal number of clusters determined using silhouette method
- Jaccard stability index computed via 100 bootstrap iterations to assess cluster robustness
- Cluster composition compared to formal party labels to identify data-driven ideological groupings

**Predictive Modeling (No Data Leakage)**
- Nested cross-validation framework: outer 10-fold CV repeated 5 times, inner resampling for hyperparameter tuning
- Multiple algorithms evaluated: random forest, logistic regression, k-nearest neighbors
- Random forest selected as best-performing classifier for predicting party from 26 questions
- Performance distributions across CV folds summarized and statistically compared between models

**Feature Importance Analysis (Out-of-Sample)**
- Bootstrap-based importance: computed within resampling framework (50% weight in consensus)
- Cross-validated permutation importance: accuracy drop when each feature shuffled (40% weight)
- SHAP values: computed on full-data model for interpretation only (10% weight, not used for selection)
- Consensus ranking: normalized importance scores combined across methods to identify top discriminating questions
- Top 10 questions with highest consensus scores selected for network construction and article topic alignment

**Network Analysis**
- Direction-agreement networks constructed based on pairwise sign concordance (positive/negative opinion alignment)
- Networks built for full 26-question set and top-10 subset using agreement threshold of 0.7
- Edge weights represent proportion of questions on which candidate pairs agree in direction
- Community detection and visualization performed to identify ideological clusters

---

