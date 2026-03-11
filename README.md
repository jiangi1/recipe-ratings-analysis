# Rating the Recipe: A Data Story

*Isabella Jiang*

## Introduction

This project analyzes recipe data from Food.com to understand what factors influence recipe ratings. The dataset contains **231,637** unique recipes and **1,071,520** user reviews after cleaning.

**Main Question:** Do sugary recipes receive lower ratings than non-sugary recipes?

**Dataset Size:** **1,132,367** rows after cleaning.

**Key Columns:**
- `rating`: User rating (1-5 stars)
- `prop_sugar`: Proportion of calories from sugar
- `is_dessert`: Whether recipe is a dessert
- `minutes`: Cooking time
- `calories`: Total calories

## Data Cleaning and Exploratory Data Analysis

### Data Cleaning
I performed several cleaning steps:
- Replaced 0 ratings with NaN (0 means missing)
- Split the nutrition column into separate columns for calories, sugar, fat, etc.
- Created `prop_sugar` to measure sugar content as a proportion of total calories
- Added `is_dessert` flag by checking if 'dessert' appears in recipe tags
- Calculated `average_rating` per recipe for analysis

### Univariate Analysis

**Distribution of Ratings:**
Ratings are heavily left-skewed, with most recipes receiving 4 or 5 stars. Very few recipes receive ratings below 3 stars.

<iframe src="assets/rating_distribution.html" width="100%" height="500" frameborder="0"></iframe>

**Distribution of Sugar Proportion:**
Most recipes have low sugar content, with a long tail of sugary recipes. The distribution is right-skewed, indicating that sugary recipes are less common.

<iframe src="assets/prop_sugar_distribution.html" width="100%" height="500" frameborder="0"></iframe>

### Bivariate Analysis

**Ratings by Dessert Type:**
Dessert recipes show slightly lower ratings than non-dessert recipes (4.65 vs 4.66), despite having higher sugar content (36% vs 13%).

<iframe src="assets/ratings_by_dessert.html" width="100%" height="500" frameborder="0"></iframe>

**Rating vs. Sugar Proportion:**
There is a weak relationship between sugar content and ratings, with very sugary recipes showing slightly more variation in ratings.

<iframe src="assets/rating_vs_sugar.html" width="100%" height="500" frameborder="0"></iframe>

**Ratings Over Time:**
Average recipe ratings have remained relatively stable over the years, with a slight downward trend in recent years.

<iframe src="assets/ratings_over_time.html" width="100%" height="500" frameborder="0"></iframe>

### Interesting Aggregates

The table below shows how dessert and non-dessert recipes compare across key metrics:

| is_dessert | rating_mean | rating_median | rating_count | prop_sugar_mean | calories_mean |
|------------|-------------|---------------|--------------|-----------------|---------------|
| False | 4.66 | 5.0 | 881,703 | 0.13 | 439.0 |
| True | 4.65 | 5.0 | 189,817 | 0.36 | 556.8 |

Dessert recipes have significantly higher sugar content (36% of calories vs. 13%) and more calories, but their average ratings are only slightly lower than non-dessert recipes.

The relationship between sugar content and ratings becomes clearer when looking at sugar quartiles:

<iframe src="assets/rating_by_sugar_quartile.html" width="100%" height="500" frameborder="0"></iframe>

Interestingly, recipes in the highest sugar quartile actually receive the highest ratings (4.70 for non-dessert, 4.66 for dessert), suggesting that sugar might be associated with better-tasting recipes.

## Assessment of Missingness

### MNAR Analysis
The **'review'** column is likely **Missing Not at Random (MNAR)** because people tend to write reviews only when they have strong opinions (loved or hated a recipe). Users with neutral reactions are much less likely to take time to write a review. This means the missingness itself tells us something about the unobserved value - the reviewer probably felt indifferent.

To make this MAR, we would need additional data like user engagement metrics, whether they saved the recipe, or their review history that could explain review-writing behavior.

### Missingness Dependency

I tested whether missing ratings depend on other columns using permutation tests. About **5.4%** of ratings are missing in the dataset.

**Test 1: Does rating missingness depend on sugar content?**
- **P-value:** 0.0000
- **Conclusion:** Missingness **DOES depend** on sugar content. Recipes with missing ratings tend to have different sugar proportions than those with ratings.

**Test 2: Does rating missingness depend on cooking time?**
- **P-value:** 1.0000
- **Conclusion:** Missingness does **NOT depend** on cooking time. The missingness of ratings is independent of how long a recipe takes to prepare.

The visualization below shows how sugar content differs between recipes with missing vs. present ratings:

<iframe src="assets/missing_plot.html" width="100%" height="500" frameborder="0"></iframe>

## Hypothesis Testing

**Question:** Do people rate sugary recipes differently than non-sugary recipes?

- **Null Hypothesis:** The mean rating of sugary recipes equals the mean rating of non-sugary recipes.
- **Alternative Hypothesis:** The mean rating of sugary recipes is different from the mean rating of non-sugary recipes.
- **Test Statistic:** Absolute difference in means |mean rating of sugary recipes - mean rating of non-sugary recipes|
- **Significance Level:** 0.05

**Results:**
- Mean rating (sugary recipes): 4.669
- Mean rating (non-sugary recipes): 4.657
- Observed difference: 0.0120
- **P-value:** 0.0000

**Conclusion:** Since the p-value (0.0000) is less than 0.05, I **reject the null hypothesis**. There is statistically significant evidence that sugary recipes receive different ratings than non-sugary recipes. Interestingly, sugary recipes actually receive **slightly higher** average ratings (4.669 vs. 4.657), contrary to my initial expectation that they might be rated lower.

## Framing a Prediction Problem

I built a model to predict a recipe's **average rating** as a continuous value.

- **Type:** Regression
- **Response Variable:** `average_rating` (continuous, ranging from 1-5)
- **Metric:** RMSE (Root Mean Square Error) - measures average prediction error in the same units as ratings
- **Why RMSE:** It provides an interpretable measure of how far off predictions are, on average, from the true rating
- **Time of Prediction:** All recipe features (ingredients, cooking time, nutrition, etc.) are known before any ratings are given, so they are valid to use for prediction

## Baseline Model

**Features used:**
- `prop_sugar` (quantitative): Proportion of calories from sugar
- `is_dessert` (nominal): Whether the recipe is a dessert (one-hot encoded)

**Model:** A simple approach of predicting the mean rating from the training data for all recipes.

**Performance:**
- **Train RMSE:** 0.2596
- **Validation RMSE:** 0.2589
- **Test RMSE:** 0.2610
- **R²:** ~0.00 (essentially zero)

The baseline model performs no better than simply guessing the average rating for every recipe. This is expected since it uses only two simple features and cannot capture the complexity of what makes recipes highly rated.

## Final Model

**New Engineered Features:**
- `calories_norm`: Normalized calories (calories divided by mean calories)
- `log_minutes`: Log-transformed cooking time to handle skewness
- `steps_per_ingredient`: Recipe complexity (number of steps divided by number of ingredients)
- `is_high_sugar`: Flag for high sugar content (1 if prop_sugar > 0.2, else 0)

**Model:** Manual random forest regressor with hyperparameter tuning

**Hyperparameter Tuning:** Grid search over:
- Number of trees: 5, 10, 15, 20
- Max depth: 3, 5, 7, 10
- Min samples split: 3, 5, 7, 10

**Best Parameters:** 
- `n_trees`: 15
- `max_depth`: 7
- `min_samples_split`: 7

**Performance:**
- **Test RMSE:** 0.2609
- **Test R²:** 0.0007
- **Improvement:** RMSE decreased by 0.0001 compared to baseline

The final model shows a slight improvement over the baseline, though the R² value near zero indicates that predicting recipe ratings is challenging with the available features.

### Feature Importance

The plot below shows which features were most important in the final model. Interestingly, all features have nearly equal importance (0.167 each), suggesting that no single factor dominates recipe ratings:

<iframe src="assets/feature_importance.html" width="100%" height="500" frameborder="0"></iframe>

### Predictions vs. Actual

The scatter plot compares predicted ratings to actual ratings. Points closer to the diagonal line indicate better predictions:

<iframe src="assets/predictions_scatter.html" width="100%" height="500" frameborder="0"></iframe>

### Model Comparison

<iframe src="assets/model_comparison.html" width="100%" height="500" frameborder="0"></iframe>

## Fairness Analysis

I tested whether the model performs equally well for high-calorie vs. low-calorie recipes.

- **Groups:** Low calorie (≤311.5 calories) vs. High calorie (>311.5 calories)
- **Metric:** RMSE (Root Mean Square Error)
- **Null Hypothesis:** RMSE is equal for low-calorie and high-calorie recipes
- **Alternative Hypothesis:** RMSE is higher for low-calorie recipes (model performs worse)
- **Test Statistic:** Difference in RMSE (low calorie - high calorie)
- **Significance Level:** 0.05

**Results:**
- RMSE for low-calorie recipes: 0.2615
- RMSE for high-calorie recipes: 0.2603
- Observed difference: 0.0013
- **P-value:** 0.3010

**Conclusion:** Since the p-value (0.3010) is greater than 0.05, I **fail to reject the null hypothesis**. This suggests the model is **fair** across calorie groups, with similar prediction error for both low-calorie and high-calorie recipes.

<iframe src="assets/fairness_plot.html" width="100%" height="500" frameborder="0"></iframe>

<iframe src="assets/fairness_perm.html" width="100%" height="500" frameborder="0"></iframe>

