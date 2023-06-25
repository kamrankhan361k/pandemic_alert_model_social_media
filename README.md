# Overview 
The objective of this paper is to develop an early alert model using digital data traces (Google Trends and Twitter) for identifying COVID-19 outbreaks in Germany as a case study.

The work conatains three parts:
1. Generation of symptom corpus and preparation of Google Trends/Twitter longitudinal datasets with multidimentional symptom features.
2. Log-linear regression model for up-/ down-trend analysis.
3. Random Forest and LSTMs for up-/ down-trend forecasting.

## Generate knowledge graph to get the symptom corpus.

input data: symptoms from symptom ontology (https://www.ebi.ac.uk/ols/ontologies/symp) 
convert .owl to .json: (http://vowl.visualdataweb.org/webvowl-old/webvowl-old.html)
SYMP_ONTOLOGY = "data/raw/symp.json"

output data: 
the top German ymptoms from hypergeometric test with low p_value and high volume of co-occurances in SCAIView knowledge software (https://academia.scaiview.com/).
(/data/Knowledge_graph/COVID/COVID_symptoms_from_hypergeometrictest.json)

#### 1. Request SCAIView and get symptom_disease related IDs (CPU running time: 1h 55 min)
    python3 knowledge_graph.py get_symptom_disease_IDs COVID
#### 2. Get the number of document counts of each disease
    python3 knowledge_graph.py get_disease_count
#### 3. Get the number of document counts of each symptom (CPU running time: 1h 36 min)
    python3 knowledge_graph.py get_symptoms_count
#### 4. Get disease symptoms dict with corresponding p_values from hypergeometric test
    python3 knowledge_graph.py perform_disease_hypergeo_test COVID 0.05 -v
#### 5. Get top disease related symptoms (COVID)
    python3 knowledge_graph.py get_top_relevant_symptoms 0.05 50
#### 6. Plot the symptoms with descending co-occurances with COVID-19 in PubMed/PMC.
    python3 knowledge_graph.py show_plot COVID_sort_pvalue_occurances.csv 25
#### 7. Symptom translation
    use DeepL software to translate the top English symptoms from Knowledge graph into German.
    Note: here we give an example of the german terms we retrieved till June 2022. If you translate the terms into France OR retrieve new data, you should replace the file with the route:
    (SCAIVIEW_SYMPTOM = "/data/processed/symptom_translations.csv") 
#### 8. Get German symptom terms
    input: the translated German symptoms.
    output: .json file contains the German symptom corpus.
    python3 knowledge_graph.py get_covid_symptoms

## Social media and gold standard longitudinal datasets
#### 1. Download Gold Standard data from Germany RKI.
     Surveillance data can be retrieved from Robert Koch-Institut (RKI) GitHub repository (https://github.com/orgs/robert-koch-institut/repositories)
     The surveillance data from 2020-03-01 to 2022-06-28 is downloaded and saved in /data/raw/.
#### 2. Retrieve social media data from Google Trends and Twitter with the symptom queries from knowledge graph.
     Google Trends: src/scripts/google_trends.py
     Twitter: src/scripts/twitter_api.py (NEED credentials of academic Twitter developer API)
     Note: here we retieved Google and Twitter data from Jan 2020 to Jun 2022 as an example.
     and data is in 'data/processed/daily_google_german.csv' and 'data/processed/daily_twitter_german.csv'

# Trend analysis
### Background of trend analysis
- **STL decomposition to get the trend of time series raw data**(https://www.statsmodels.org/dev/examples/notebooks/generated/stl_decomposition.html)
  - For Google Trends and Twitter, STL period: 30
  - For RKI confirmed cases, deaths and hospitalization, STL period: 7
- **Log-linear regression model**:
  - window size: 14 days
  - stride: 1 day
  - alpha: 0.05
  
#### 1. Get surveillance gold standard trends generated from log-linear regression model and save trends into csv files** (flag argument: RKI_case/ RKI_death/ RKI_hospitalization; stl_number: 7/ 7/ 7)

    python3 cli_trend_analysis.py generate_gold_standard_trend RKI_case 14 7
    python3 cli_trend_analysis.py generate_gold_standard_trend RKI_death 14 7
    python3 cli_trend_analysis.py generate_gold_standard_trend RKI_hospitalization 14 7
#### 2. Get up- and down-trends of surveillance gold standard
    python3 cli_trend_analysis.py get_trends_from_gold_standard RKI_case -f
    python3 cli_trend_analysis.py get_trends_from_gold_standard RKI_death -f 
    python3 cli_trend_analysis.py get_trends_from_gold_standard RKI_hospitalization -f
#### 3. making symptom-level trend analysis (Google Trends and Twitter)
    python3 cli_trend_analysis.py generate_proxy_trend Google_Trends daily_google_german.csv 14 30
    python3 cli_trend_analysis.py generate_proxy_trend Twitter daily_twitter_german.csv 14 30
#### 4. Get evaluation metrics of individual symptom and save the .csv file (Google Trends and Twitter)
    python3 cli_trend_analysis.py generate_evaluation_metrics RKI_case Google_Trends 2022-03-01
    python3 cli_trend_analysis.py generate_evaluation_metrics RKI_hospitalization Google_Trends 2022-03-01
    python3 cli_trend_analysis.py generate_evaluation_metrics RKI_death Google_Trends 2022-03-01

    python3 cli_trend_analysis.py generate_evaluation_metrics RKI_case Twitter 2022-03-01
    python3 cli_trend_analysis.py generate_evaluation_metrics RKI_hospitalization Twitter 2022-03-01
    python3 cli_trend_analysis.py generate_evaluation_metrics RKI_death Twitter 2022-03-01
#### 5. Get top 20 symptoms (based on the result of hypergeometric test) for each digital trace (Google Trends and Twitter)
    python3 cli_trend_analysis.py get_symptoms 20 google -f
    python3 cli_trend_analysis.py get_symptoms 20 twitter -f 
#### 6. Making digital trace (Google Trends, Twitter, and Combined) trend analysis and get up- and down-trends
    python3 cli_trend_analysis.py combined_proxy 20 Google_Trends 0.05 2022-03-01 -r -f
    python3 cli_trend_analysis.py combined_proxy 20 Twitter 0.05 2022-03-01 -r -f
    python3 cli_trend_analysis.py get_combined_P_trends 0.05 2022-03-01 -f

#### 7. Get evaluation metrics for each digital trace (Google Trends, Twitter and Combined trace)
    python3 cli_trend_analysis.py generate_metrics_for_combined_proxy_or_combinedP Google_Trends RKI_case 2022-03-01
    python3 cli_trend_analysis.py generate_metrics_for_combined_proxy_or_combinedP Google_Trends RKI_death 2022-03-01
    python3 cli_trend_analysis.py generate_metrics_for_combined_proxy_or_combinedP Google_Trends RKI_hospitalization 2022-03-01

    python3 cli_trend_analysis.py generate_metrics_for_combined_proxy_or_combinedP Twitter RKI_case 2022-03-01
    python3 cli_trend_analysis.py generate_metrics_for_combined_proxy_or_combinedP Twitter RKI_death 2022-03-01
    python3 cli_trend_analysis.py generate_metrics_for_combined_proxy_or_combinedP Twitter RKI_hospitalization 2022-03-01

    python3 cli_trend_analysis.py generate_metrics_for_combined_proxy_or_combinedP Combined RKI_case 2022-03-01
    python3 cli_trend_analysis.py generate_metrics_for_combined_proxy_or_combinedP Combined RKI_death 2022-03-01
    python3 cli_trend_analysis.py generate_metrics_for_combined_proxy_or_combinedP Combined RKI_hospitalization 2022-03-01

#### 8. Visualization of trends of surveillance data and up-/down-trends of Google Trends and Combined trace
    python3 cli_trend_analysis.py visualize_trend
#### 9. Pairwise event visualization
    python3 cli_trend_analysis.py plot_pairwise_trend_event RKI_case Up_trends 2020_2022 2022-03-01
    python3 cli_trend_analysis.py plot_pairwise_trend_event RKI_case Down_trends 2020_2022 2022-03-01
    python3 cli_trend_analysis.py plot_pairwise_trend_event RKI_death Up_trends 2020_2022 2022-03-01
    python3 cli_trend_analysis.py plot_pairwise_trend_event RKI_death Down_trends 2020_2022 2022-03-01
    python3 cli_trend_analysis.py plot_pairwise_trend_event RKI_hospitalization Up_trends 2020_2022 2022-03-01
    python3 cli_trend_analysis.py plot_pairwise_trend_event RKI_hospitalization Down_trends 2020_2022 2022-03-01

    * This function will also print out the percentage of onsets of up-trends.

--------
# Trend forecasting
## Random Forest
Note: The forecasting horizon is set based on the result of trend analysis. We set the timepoints to split train/test sets.
training length: 28 days
forecasting horizon: consistent with the time interval in log-linear regression model: 14 days

#### 1. Prepare dataset for all feature space (Google Trends/Combined)
    Note: proxy: Google; Combined
          gold_standard: RKI_case; RKI_hospitalization

    python3 RF_data_preprocessing.py --proxy=Google --gold_standard=RKI_case --time_start='2020-03-01' --time_end='2022-06-15' --split_date='2022-04-01' --forecasting_horizon=14 --training_length=28

    python3 RF_data_preprocessing.py --proxy=Google --gold_standard=RKI_hospitalization --time_start='2020-03-01' --time_end='2022-06-15' --split_date='2022-04-01' --forecasting_horizon=14 --training_length=28

    python3 RF_data_preprocessing.py --proxy=Combined --gold_standard=RKI_case --time_start='2020-03-01' --time_end='2022-06-15' --split_date='2022-04-01' --forecasting_horizon=14 --training_length=28

    python3 RF_data_preprocessing.py --proxy=Combined --gold_standard=RKI_hospitalization --time_start='2020-03-01' --time_end='2022-06-15'  --split_date='2022-04-01' --forecasting_horizon=14 --training_length=28

#### 2. Run Random Forest models
    Note: proxy: Google; Combined
          gold_standard: RKI_case; RKI_hospitalization

    python3 Random_Forest_optuna.py --proxy=Google --gold_standard=RKI_case --forecasting_horizon=14 --training_length=28 --number_trial=90 --cv_initial_window=90 --cv_step_length=70 --cv_test_window=30

    python3 Random_Forest.py --proxy=Google --gold_standard=RKI_case --forecasting_horizon=14 --mode=train --n_estimators=*** --max_depth=*** --min_samples_split=*** --min_samples_leaf=*** --max_features=***

    python3 Random_Forest.py --proxy=Google --gold_standard=RKI_case --forecasting_horizon=14 --mode=test

#### 3. Run LSTMs
    Note: type: Google_confirmed_cases; Google_hospitalization; Combined_confirmed_cases; Combined_hospitalization

    python3 LSTM_train_optuna.py --type=Google_confirmed_cases --forecasting_horizon=14 --tranining_length=28 --mode=train --GPU=0 --testset_split=2022-04-27

    python3 LSTM_train.py --type=Google_confirmed_cases --forecasting_horizon=14 --tranining_length=28 --GPU=0 --testset_split=2022-04-27

All source code that is specific to this project.

<p><small>Project based on the <a target="_blank" href="https://drivendata.github.io/cookiecutter-data-science/">cookiecutter data science project template</a>. #cookiecutterdatascience</small></p>
  






