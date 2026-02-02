---
title: 'Building Efficient Data Pipelines with Polars: A Step-by-Step Guide'
date: 2024-12-09
permalink: /posts/2024/12/polars-data-pipeline/
tags:
  - polars
  - python
  - data-engineering
---

Data pipelines are at the heart of modern data engineering. They allow you to automate data ingestion, cleaning, transformation, and feature generation—making your datasets ready for analysis or machine learning tasks.

In this post, we’ll explore how to build **scalable, modular, and efficient data pipelines using Polars**. Along the way, you’ll learn best practices like using lazy evaluation, chaining operations with .pipe(), and structuring your project for clarity and maintainability.

We’ll be working with a [YouTube trending videos dataset](https://www.kaggle.com/datasets/datasnaek/youtube-new?resource=download&sort=published) (public domain, available on Kaggle) as our example.

## Setup
Before we start, ensure you have Polars installed and up to date:. We will **uv** for python dependencies management. You can install it first using the [official documentation](https://docs.astral.sh/uv/guides/install-python/)

```bash
uv init
uv add polars
```

## What is a Data Pipeline?
A data pipeline is a sequence of automated steps that:

1. Pull data from one or more sources
1. Transform and clean the data
1. Save the processed dataset for downstream tasks

With Polars, building pipelines becomes elegant and efficient thanks to method chaining and lazy evaluation.

## Step 0: Utility Function

```python
def read_category_mappings(path: str) -> Dict[int, str]:
    """Load and parse YouTube category mappings from JSON."""
    with open(path, "r") as f:
        categories = json.load(f)
    return {int(c["id"]): c["snippet"]["title"] for c in categories["items"]}

```

**Explanation:**

- This function reads a JSON file containing YouTube category information.

- The JSON structure typically looks like this:

```json
{
  "items": [
    {"id": "1", "snippet": {"title": "Film & Animation"}},
    {"id": "2", "snippet": {"title": "Autos & Vehicles"}}
  ]
}

```

It returns a dictionary mapping category IDs (as integers) to human-readable names:

```python
{1: "Film & Animation", 2: "Autos & Vehicles"}
```

## Step 1: Load Data

```python
def load_data(videos_path: str, categories_path: str):
    id_to_category = read_category_mappings(categories_path)
    df = pl.read_csv(videos_path)
    return df, id_to_category

```

**Explanation**:

- Reads the CSV file with video metadata (GBvideos.csv) into a Polars DataFrame.

- Calls read_category_mappings() to get the mapping dictionary.

- Returns both the DataFrame and the category mapping.

- Essentially, this is the entry point for raw data into the pipeline.

## Step 2: Parse Dates

```python
def parse_dates(df: pl.DataFrame) -> pl.DataFrame:
    return df.with_columns([
        pl.col("trending_date").str.to_datetime("%y.%d.%m").alias("trending_date_dt"),
        pl.col("publish_time").str.to_datetime("%Y-%m-%dT%H:%M:%S%.fZ").alias("publish_time_dt"),
    ])


```

**Explanation:**

- Converts string representations of dates into Polars datetime objects.

- trending_date is in YY.DD.MM format (strange YouTube format).

- publish_time is in ISO format YYYY-MM-DDTHH:MM:SSZ.

- Why: This enables date arithmetic like calculating how many days it took for a video to trend.

## Step 3: Add Category Names

```python

def add_categories(df: pl.DataFrame, id_to_category: Dict[int, str]) -> pl.DataFrame:
    return df.with_columns([
        pl.col("category_id").replace_strict(id_to_category, default="Unknown").alias("category")
    ])


```
**Explanation**:

- Replaces numeric category_id in the video DataFrame with human-readable category names using the dictionary from Step 1.

- If a category ID is missing in the mapping, it defaults to "Unknown".

- This is helpful for readability and analysis (e.g., grouping by category).

## Step 4: Compute Engagement Metrics

```python
def compute_engagement_metrics(df: pl.DataFrame) -> pl.DataFrame:
    return df.with_columns([
        (pl.col("likes") / pl.col("dislikes").replace(0, None)).alias("likes_to_dislikes"),
        (pl.col("likes") / pl.col("views")).alias("likes_to_views"),
        (pl.col("comment_count") / pl.col("views")).alias("comments_to_views"),
    ])

```

**Explanation:**

- Calculates ratios to measure engagement:

- likes_to_dislikes: Positive reaction vs negative reaction.

- likes_to_views: Likes normalized by views.

- comments_to_views: Comments normalized by views.

- Handles divide-by-zero safely by replacing zero dislikes with None.

## Step 5: Compute Temporal Metrics

```python
def compute_temporal_metrics(df: pl.DataFrame) -> pl.DataFrame:
    return df.with_columns([
        (pl.col("trending_date_dt") - pl.col("publish_time_dt"))
        .dt.total_days()
        .alias("days_to_trending"),
        pl.col("trending_date_dt").dt.weekday().alias("trending_weekday")
    ])

```

**Explanation:**

- Computes how long it took for a video to trend (days_to_trending).

- Extracts the weekday of trending (trending_weekday) to see patterns like whether videos trend more on certain days.

- Uses the datetime columns created in Step 2.

## Step 6: Aggregate Per Video

```python
def aggregate_videos(df: pl.DataFrame) -> pl.DataFrame:
    return (
        df.group_by("video_id")
        .agg([
            (pl.col("trending_date_dt").max() - pl.col("trending_date_dt").min())
            .dt.total_days()
            .alias("days_in_trending"),
            pl.col("views").first().alias("initial_views"),
            pl.col("likes").first().alias("initial_likes"),
            pl.col("category").first().alias("category"),
            pl.col("days_to_trending").min().alias("min_days_to_trend"),
            pl.col("likes_to_dislikes").mean().alias("avg_likes_ratio"),
            pl.len().alias("trending_appearances")
        ])
        .sort("days_in_trending", descending=True)
    )

```

**Explanation:**

- Groups data by video_id because a video can appear on trending multiple times.

- Aggregates key metrics per video:

- days_in_trending: Total days a video stayed trending.

- initial_views & initial_likes: Values when the video first appeared.

- category: First category (same for all rows of a video).

- min_days_to_trend: Minimum days from publish to trending.

- avg_likes_ratio: Average likes/dislikes across appearances.

- trending_appearances: How many times it appeared on trending.

- Sorts by days_in_trending to see the most trending videos first.

## Main Function

```python
def main():
    # Paths
    videos_path = "GBvideos.csv"
    categories_path = "GB_category_id.json"

    # Run pipeline
    df, id_to_category = load_data(videos_path, categories_path)
    df = parse_dates(df)
    df = add_categories(df, id_to_category)
    df = compute_engagement_metrics(df)
    df = compute_temporal_metrics(df)
    target = aggregate_videos(df)

    # Outputs
    print(f"Processed {len(target)} unique trending videos")
    print(f"Dataset shape: {target.shape}")
    print("\nTop 5 videos by days in trending:")
    print(target.head())

```

**Explanation:**

- Orchestrates the entire pipeline:

- Loads raw data.

- Parses dates for calculations.

- Maps category names.

- Computes engagement metrics.

- Computes temporal metrics.

- Aggregates results per video.

- Prints summary statistics and top results.

Using ```python if  __name__ == "__main__": main() ``` ensures the pipeline runs only when the script is executed, not when imported.

You can get the entire source code via [github](https://github.com/franklinokech/polar-data-pipeline)