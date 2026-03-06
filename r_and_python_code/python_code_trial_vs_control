# GOAL: Import libraries for data work, stats, plotting, and PDF export.
# Import pandas for reading CSVs and working with tables.
import pandas as pd
# Import numpy for math and missing value handling.
import numpy as np
# Import matplotlib for plotting.
import matplotlib.pyplot as plt
# Import PdfPages for saving multiple figures into one PDF.
from matplotlib.backends.backend_pdf import PdfPages
# Import scipy stats for t distribution critical values.
from scipy import stats

# GOAL: Set simple display and plotting defaults.
# Show more columns when printing tables.
pd.set_option("display.max_columns", 80)
# Make printed tables wider before wrapping.
pd.set_option("display.width", 180)
# Use a clean default plot style.
plt.style.use("default")

# GOAL: Load CSV File
# Load the Task 2 dataset provided by Forage
data = pd.read_csv("QVI_data.csv")

# GOAL: Clean and standardize column names so references are consistent.
# Remove leading or trailing spaces
data.columns = [c.strip() for c in data.columns]
# Convert Task 2 dataset column names to uppercase for consistent references.
data = data.rename(columns={c: c.upper() for c in data.columns})

# Checking to see data types
print(data[["STORE_NBR","LYLTY_CARD_NBR","TXN_ID","TOT_SALES","PROD_QTY"]].dtypes)

# GOAL: Remove rows with missing values
data = data.dropna(subset=["DATE", "STORE_NBR", "LYLTY_CARD_NBR", "TXN_ID", "TOT_SALES", "PROD_QTY"]).copy()


# GOAL: Create YEARMONTH (yyyymm) from DATE so we can do monthly store analysis like the R template.
# Convert DATE to datetime (your DATE is already like 2018-11-02, so this is straightforward).
data["DATE"] = pd.to_datetime(data["DATE"], errors="coerce")
# Drop rows where DATE could not be converted.
data = data.dropna(subset=["DATE"]).copy()
# Create YEARMONTH as an integer string YYYYMM then convert to int.
data["YEARMONTH"] = data["DATE"].dt.strftime("%Y%m").astype(int)

# GOAL: Build the monthly measures per store that the template uses for control store selection.
# Group the dataset by store and yearmonth.
g = data.groupby(["STORE_NBR", "YEARMONTH"], as_index=False)
# Aggregate monthly totals and counts needed for the analysis.
measureOverTime = g.agg(
    totSales=("TOT_SALES", "sum"),
    nCustomers=("LYLTY_CARD_NBR", pd.Series.nunique),
    nTransactions=("TXN_ID", pd.Series.nunique),
    totChips=("PROD_QTY", "sum"),
)




# Create transactions per customer per store-month.
measureOverTime["nTxnPerCust"] = measureOverTime["nTransactions"] / measureOverTime["nCustomers"]
# Create chips per transaction per store-month.
measureOverTime["nChipsPerTxn"] = measureOverTime["totChips"] / measureOverTime["nTransactions"]
# Create average price per unit per store-month.
measureOverTime["avgPricePerUnit"] = measureOverTime["totSales"] / measureOverTime["totChips"]
# Sort for readability.
measureOverTime = measureOverTime.sort_values(["STORE_NBR", "YEARMONTH"]).reset_index(drop=True)



# GOAL: Define trial and pre-trial months
# Set the first month of the trial period (Feb 2019).
TRIAL_START = 201902
# Set the last month of the trial period (Apr 2019).
TRIAL_END = 201904
# Set the first month that is NOT pre-trial (Feb 2019 starts trial).
PRETRIAL_END = 201902

# GOAL: Filter to stores that have data for every pre-trial month that exists
# Count how many unique pre-trial months exist in the dataset overall
required_pretrial_months = measureOverTime.loc[measureOverTime["YEARMONTH"] < PRETRIAL_END, "YEARMONTH"].nunique()
# Count how many distinct pre-trial months each store has.
pretrial_month_counts = (
    measureOverTime.loc[measureOverTime["YEARMONTH"] < PRETRIAL_END]
    .groupby("STORE_NBR")["YEARMONTH"]
    .nunique()
)



# Keep only stores that appear in all available pre-trial months.
storesWithFullObs = pretrial_month_counts[pretrial_month_counts == required_pretrial_months].index
# Build the pre-trial measures table for control selection.
preTrialMeasures = measureOverTime.loc[
    (measureOverTime["YEARMONTH"] < PRETRIAL_END) & (measureOverTime["STORE_NBR"].isin(storesWithFullObs))
].copy()

# GOAL: Create a helper to convert YEARMONTH (yyyymm) into a real datetime for plotting.
# Define a function to convert an integer like 201902 into a datetime like 2019-02-01.
def yearmonth_to_date(yearmonth_series: pd.Series) -> pd.Series:
    # Convert yearmonth integers to strings.
    s = yearmonth_series.astype(str)
    # Create a date string like YYYY-MM-01.
    s = s.str.slice(0, 4) + "-" + s.str.slice(4, 6) + "-01"
    # Convert the strings into datetime.
    return pd.to_datetime(s, errors="coerce")


# GOAL: Create the correlation function (template section: calculateCorrelation).
# Define a function that correlates one metric for a trial store against each candidate control store.
def calculateCorrelation(inputTable: pd.DataFrame, metricCol: str, storeComparison: int) -> pd.DataFrame:
    # Copy the input table so we do not accidentally modify the original.
    df = inputTable.copy()
    # Get the list of store numbers in the input.
    store_numbers = df["STORE_NBR"].unique()
    # Create the trial store time series for the chosen metric.
    trial_series = df.loc[df["STORE_NBR"] == storeComparison, ["YEARMONTH", metricCol]].rename(columns={metricCol: "trial"})
    # Create an empty list to store the output rows.
    rows = []
    # Loop through each store as a possible control store.
    for s in store_numbers:
        # Skip the trial store itself.
        if s == storeComparison:
            continue
        # Create the candidate store time series for the chosen metric.
        cand_series = df.loc[df["STORE_NBR"] == s, ["YEARMONTH", metricCol]].rename(columns={metricCol: "cand"})
        # Align trial and candidate months so the correlation uses the same months.
        merged = trial_series.merge(cand_series, on="YEARMONTH", how="inner")
        # Skip if we do not have enough months to compute correlation.
        if len(merged) < 2:
            continue
        # Compute Pearson correlation between trial and candidate values.
        corr_val = merged["trial"].corr(merged["cand"])
        # Append one result row for this candidate store.
        rows.append({"Store1": storeComparison, "Store2": int(s), "corr_measure": corr_val})
    # Return results as a DataFrame.
    return pd.DataFrame(rows)

# GOAL: Create the magnitude distance function
# Define a function that measures similarity based on standardized absolute differences.
def calculateMagnitudeDistance(inputTable: pd.DataFrame, metricCol: str, storeComparison: int) -> pd.DataFrame:
    # Copy the input table so we do not accidentally modify the original.
    df = inputTable.copy()
    # Get the list of store numbers in the input.
    store_numbers = df["STORE_NBR"].unique()
    # Extract the trial store metric values by month.
    trial = df.loc[df["STORE_NBR"] == storeComparison, ["YEARMONTH", metricCol]].rename(columns={metricCol: "trial"})
    # Create a list to store month-level absolute differences for each store.
    diffs = []
    # Loop through each candidate store.
    for s in store_numbers:
        # Skip the trial store itself.
        if s == storeComparison:
            continue
        # Extract candidate store metric values by month.
        cand = df.loc[df["STORE_NBR"] == s, ["YEARMONTH", metricCol]].rename(columns={metricCol: "cand"})
        # Align trial and candidate months.
        merged = trial.merge(cand, on="YEARMONTH", how="inner")
        # Compute absolute difference between trial and candidate for each month.
        merged["measure"] = (merged["trial"] - merged["cand"]).abs()
        # Add the trial store identifier.
        merged["Store1"] = storeComparison
        # Add the candidate store identifier.
        merged["Store2"] = int(s)
        # Keep only the columns needed for standardization.
        diffs.append(merged[["Store1", "Store2", "YEARMONTH", "measure"]])
    # Combine all candidate diffs into one table.
    calcDistTable = pd.concat(diffs, ignore_index=True)
    # Compute min and max distance across candidates for each trial store and month.
    minMaxDist = calcDistTable.groupby(["Store1", "YEARMONTH"], as_index=False).agg(
        minDist=("measure", "min"),
        maxDist=("measure", "max"),
    )
    # Merge min and max back onto each candidate row.
    distTable = calcDistTable.merge(minMaxDist, on=["Store1", "YEARMONTH"], how="left")
    # Create a safe denominator for standardization (avoid division by zero).
    denom = (distTable["maxDist"] - distTable["minDist"]).replace(0, np.nan)
    # Standardize to a similarity score from 0 to 1 (higher means more similar).
    distTable["magnitudeMeasure"] = 1 - (distTable["measure"] - distTable["minDist"]) / denom
    # If denom was zero, treat similarity as 1.0 for that month.
    distTable["magnitudeMeasure"] = distTable["magnitudeMeasure"].fillna(1.0)
    # Average the standardized similarity across months for each candidate store.
    finalDistTable = distTable.groupby(["Store1", "Store2"], as_index=False).agg(
        mag_measure=("magnitudeMeasure", "mean")
    )
    # Return the magnitude similarity table.
    return finalDistTable

# GOAL: Combine correlation and magnitude scores to rank control stores (template section: combined score).
# Define a function that creates the final control ranking table for a trial store.
def rank_control_stores(preTrial: pd.DataFrame, trial_store: int, corr_weight: float = 0.5) -> pd.DataFrame:
    # Compute correlation scores for total sales.
    corr_sales = calculateCorrelation(preTrial, "totSales", trial_store)
    # Compute correlation scores for number of customers.
    corr_cust = calculateCorrelation(preTrial, "nCustomers", trial_store)
    # Compute magnitude similarity scores for total sales.
    mag_sales = calculateMagnitudeDistance(preTrial, "totSales", trial_store)
    # Compute magnitude similarity scores for number of customers.
    mag_cust = calculateMagnitudeDistance(preTrial, "nCustomers", trial_store)

    # Merge correlation and magnitude for sales.
    score_sales = corr_sales.merge(mag_sales, on=["Store1", "Store2"], how="inner")
    # Compute combined sales score.
    score_sales["scoreNSales"] = corr_weight * score_sales["corr_measure"] + (1 - corr_weight) * score_sales["mag_measure"]

    # Merge correlation and magnitude for customers.
    score_cust = corr_cust.merge(mag_cust, on=["Store1", "Store2"], how="inner")
    # Compute combined customer score.
    score_cust["scoreNCust"] = corr_weight * score_cust["corr_measure"] + (1 - corr_weight) * score_cust["mag_measure"]

    # Merge sales and customer scores into one table.
    score_control = score_sales[["Store1", "Store2", "scoreNSales"]].merge(
        score_cust[["Store1", "Store2", "scoreNCust"]],
        on=["Store1", "Store2"],
        how="inner",
    )
    # Compute final combined score as an average of the two driver scores.
    score_control["finalControlScore"] = 0.5 * score_control["scoreNSales"] + 0.5 * score_control["scoreNCust"]
    # Sort by best score first.
    score_control = score_control.sort_values("finalControlScore", ascending=False).reset_index(drop=True)
    # Return the ranking table.
    return score_control

# GOAL: Create a plot to visually compare trial vs control vs other stores in the pre-trial window (template visual checks).
# Define a function to build a pre-trial trend plot for one metric.
def plot_pretrial_trends(measures: pd.DataFrame, trial_store: int, control_store: int, metric: str, y_label: str, title: str) -> plt.Figure:
    # Copy measures so we can add columns safely.
    df = measures.copy()
    # Label each row as Trial, Control, or Other stores.
    df["Store_type"] = np.where(df["STORE_NBR"] == trial_store, "Trial", np.where(df["STORE_NBR"] == control_store, "Control", "Other stores"))
    # Compute the mean metric value by month and store type (Other stores becomes an average line).
    df = df.groupby(["YEARMONTH", "Store_type"], as_index=False)[metric].mean()
    # Convert YEARMONTH to a datetime month for plotting.
    df["TransactionMonth"] = yearmonth_to_date(df["YEARMONTH"])
    # Filter to the pre-trial visual check window used in the template (up to Feb 2019 inclusive of Jan, so < 201903).
    df = df.loc[df["YEARMONTH"] < 201903].copy()
    # Create a new figure.
    fig = plt.figure()
    # Plot each store type as a separate line.
    for t in ["Trial", "Control", "Other stores"]:
        # Filter to one store type.
        sub = df.loc[df["Store_type"] == t]
        # Plot that store type line.
        plt.plot(sub["TransactionMonth"], sub[metric], label=t)
    # Set the x-axis label.
    plt.xlabel("Month of operation")
    # Set the y-axis label.
    plt.ylabel(y_label)
    # Set the plot title.
    plt.title(title)
    # Show legend.
    plt.legend()
    # Return the figure.
    return fig

# GOAL: Assess trial impact for a metric by scaling control store and building confidence bands (template trial assessment).
# Define a function to run the scaling, percent difference, t-values, and plot.
def assess_trial_metric(measures: pd.DataFrame, trial_store: int, control_store: int, metric: str, y_label: str, title: str):
    # Filter to just trial and control stores.
    sub = measures.loc[measures["STORE_NBR"].isin([trial_store, control_store])].copy()

    # Compute pre-trial total for trial store for scaling.
    trial_pre_total = sub.loc[(sub["STORE_NBR"] == trial_store) & (sub["YEARMONTH"] < PRETRIAL_END), metric].sum()
    # Compute pre-trial total for control store for scaling.
    control_pre_total = sub.loc[(sub["STORE_NBR"] == control_store) & (sub["YEARMONTH"] < PRETRIAL_END), metric].sum()
    # Compute scaling factor to align control with trial baseline.
    scaling_factor = trial_pre_total / control_pre_total

    # Build scaled control series.
    control_series = sub.loc[sub["STORE_NBR"] == control_store, ["YEARMONTH", metric]].copy()
    # Create scaled control values.
    control_series["control_scaled"] = control_series[metric] * scaling_factor
    # Keep only yearmonth and scaled values.
    control_series = control_series[["YEARMONTH", "control_scaled"]]

    # Build trial series.
    trial_series = sub.loc[sub["STORE_NBR"] == trial_store, ["YEARMONTH", metric]].copy()
    # Rename trial metric column for clarity.
    trial_series = trial_series.rename(columns={metric: "trial_value"})

    # Merge trial and control on yearmonth.
    compare = trial_series.merge(control_series, on="YEARMONTH", how="inner")
    # Compute absolute percentage difference like the template.
    compare["percentageDiff"] = (compare["trial_value"] - compare["control_scaled"]).abs() / compare["control_scaled"]

    # Compute std dev of pre-trial percentage differences.
    pretrial_mask = compare["YEARMONTH"] < PRETRIAL_END
    # Calculate std dev (sample).
    stdDev = compare.loc[pretrial_mask, "percentageDiff"].std(ddof=1)

    # Compute degrees of freedom based on number of available pre-trial months minus 1.
    pretrial_months_n = compare.loc[pretrial_mask, "YEARMONTH"].nunique()
    # Set degrees of freedom.
    degreesOfFreedom = int(pretrial_months_n - 1)

    # Compute t-values for each month using (x - 0) / stdDev like the template.
    compare["tValue"] = compare["percentageDiff"] / stdDev

    # Compute the 95th percentile critical value for a one-sided test.
    t_critical_95 = stats.t.ppf(0.95, df=degreesOfFreedom)

    # Prepare a plotting table.
    plot_df = compare.copy()
    # Convert YEARMONTH to datetime month.
    plot_df["TransactionMonth"] = yearmonth_to_date(plot_df["YEARMONTH"])
    # Create series for plotting.
    plot_df["Control"] = plot_df["control_scaled"]
    # Create series for plotting.
    plot_df["Trial"] = plot_df["trial_value"]
    # Create an upper confidence band using 2 standard deviations (template approximation).
    plot_df["Control_95"] = plot_df["Control"] * (1 + stdDev * 2)
    # Create a lower confidence band using 2 standard deviations (template approximation).
    plot_df["Control_5"] = plot_df["Control"] * (1 - stdDev * 2)

    # Create figure.
    fig = plt.figure()
    # Plot trial line.
    plt.plot(plot_df["TransactionMonth"], plot_df["Trial"], label="Trial")
    # Plot scaled control line.
    plt.plot(plot_df["TransactionMonth"], plot_df["Control"], label="Control (scaled)")
    # Plot upper band.
    plt.plot(plot_df["TransactionMonth"], plot_df["Control_95"], label="Control 95% band")
    # Plot lower band.
    plt.plot(plot_df["TransactionMonth"], plot_df["Control_5"], label="Control 5% band")

    # Highlight the trial window (Feb 2019 to Apr 2019).
    trial_window_mask = (plot_df["YEARMONTH"] >= TRIAL_START) & (plot_df["YEARMONTH"] <= TRIAL_END)
    # Shade the trial period if months exist.
    if trial_window_mask.any():
        # Find start date for shading.
        xmin = plot_df.loc[trial_window_mask, "TransactionMonth"].min()
        # Find end date for shading.
        xmax = plot_df.loc[trial_window_mask, "TransactionMonth"].max()
        # Shade the trial region.
        plt.axvspan(xmin, xmax, alpha=0.15)

    # Set x-axis label.
    plt.xlabel("Month of operation")
    # Set y-axis label.
    plt.ylabel(y_label)
    # Set title.
    plt.title(title)
    # Show legend.
    plt.legend()

    # Return all useful outputs.
    return {
        "scaling_factor": scaling_factor,
        "compare_table": compare,
        "stdDev": stdDev,
        "degreesOfFreedom": degreesOfFreedom,
        "t_critical_95": t_critical_95,
        "figure": fig,
    }

# GOAL: Create a driver summary table to answer "what drove change" (customers vs transactions per customer vs price).
# Define a function that summarizes pre-trial vs trial averages for key metrics.
def driver_summary(measures: pd.DataFrame, trial_store: int, control_store: int):
    # Define the metrics to summarize for drivers.
    metrics = ["totSales", "nCustomers", "nTxnPerCust", "nChipsPerTxn", "avgPricePerUnit"]
    # Filter to trial and control rows.
    sub = measures.loc[measures["STORE_NBR"].isin([trial_store, control_store])].copy()
    # Create a period label for pre-trial vs trial.
    sub["Period"] = np.where(sub["YEARMONTH"] < PRETRIAL_END, "Pre-trial", np.where(sub["YEARMONTH"].between(TRIAL_START, TRIAL_END), "Trial", "Other"))
    # Keep only pre-trial and trial periods.
    sub = sub.loc[sub["Period"].isin(["Pre-trial", "Trial"])].copy()

    # Compute scaling factors for totSales and nCustomers (same idea as template scaling).
    trial_pre_sales = sub.loc[(sub["STORE_NBR"] == trial_store) & (sub["Period"] == "Pre-trial"), "totSales"].sum()
    control_pre_sales = sub.loc[(sub["STORE_NBR"] == control_store) & (sub["Period"] == "Pre-trial"), "totSales"].sum()
    sales_scale = trial_pre_sales / control_pre_sales

    trial_pre_cust = sub.loc[(sub["STORE_NBR"] == trial_store) & (sub["Period"] == "Pre-trial"), "nCustomers"].sum()
    control_pre_cust = sub.loc[(sub["STORE_NBR"] == control_store) & (sub["Period"] == "Pre-trial"), "nCustomers"].sum()
    cust_scale = trial_pre_cust / control_pre_cust

    # Create scaled versions for control store for the two level metrics.
    sub["totSales_scaled"] = np.where(sub["STORE_NBR"] == control_store, sub["totSales"] * sales_scale, sub["totSales"])
    sub["nCustomers_scaled"] = np.where(sub["STORE_NBR"] == control_store, sub["nCustomers"] * cust_scale, sub["nCustomers"])

    # Build a working metric list that uses scaled versions where appropriate.
    metric_map = {
        "totSales": "totSales_scaled",
        "nCustomers": "nCustomers_scaled",
        "nTxnPerCust": "nTxnPerCust",
        "nChipsPerTxn": "nChipsPerTxn",
        "avgPricePerUnit": "avgPricePerUnit",
    }

    # Create an empty list to build summary rows.
    out_rows = []
    # Loop through each driver metric.
    for m in metrics:
        # Pick the correct column name (scaled or not).
        col = metric_map[m]
        # Compute pre-trial average for trial store.
        trial_pre = sub.loc[(sub["STORE_NBR"] == trial_store) & (sub["Period"] == "Pre-trial"), col].mean()
        # Compute trial average for trial store.
        trial_trial = sub.loc[(sub["STORE_NBR"] == trial_store) & (sub["Period"] == "Trial"), col].mean()
        # Compute pre-trial average for control store.
        ctrl_pre = sub.loc[(sub["STORE_NBR"] == control_store) & (sub["Period"] == "Pre-trial"), col].mean()
        # Compute trial average for control store.
        ctrl_trial = sub.loc[(sub["STORE_NBR"] == control_store) & (sub["Period"] == "Trial"), col].mean()
        # Compute change for trial store.
        trial_change = trial_trial - trial_pre
        # Compute change for control store.
        ctrl_change = ctrl_trial - ctrl_pre
        # Compute difference in changes (simple diff in diff style).
        diff_in_change = trial_change - ctrl_change
        # Add one row to the output list.
        out_rows.append({
            "metric": m,
            "trial_pre_avg": trial_pre,
            "trial_trial_avg": trial_trial,
            "control_pre_avg": ctrl_pre,
            "control_trial_avg": ctrl_trial,
            "trial_change": trial_change,
            "control_change": ctrl_change,
            "trial_minus_control_change": diff_in_change,
        })
    # Return the summary as a DataFrame.
    return pd.DataFrame(out_rows)

# GOAL: Run the full Task 2 workflow for trial stores 77, 86, 88 and store everything for reporting.
# Create a list of trial stores from the task.
trial_stores = [77, 86, 88]
# Create an empty list to collect results for each trial store.
all_results = []

# Loop through each trial store and run control selection plus trial assessment.
for ts in trial_stores:
    # Create the control ranking table for this trial store.
    score_table = rank_control_stores(preTrialMeasures, ts, corr_weight=0.5)
    # Pick the top ranked control store (we already skip self in correlation, but we still guard just in case).
    control_store = int(score_table.loc[score_table["Store2"] != ts].iloc[0]["Store2"])

    # Create pre-trial sales trend plot.
    fig_pre_sales = plot_pretrial_trends(
        measureOverTime, ts, control_store, "totSales",
        "Total sales", f"Pre-trial trend: Total sales (Trial {ts} vs Control {control_store})"
    )
    # Create pre-trial customer trend plot.
    fig_pre_cust = plot_pretrial_trends(
        measureOverTime, ts, control_store, "nCustomers",
        "Total customers", f"Pre-trial trend: Customers (Trial {ts} vs Control {control_store})"
    )

    # Run trial assessment for sales.
    sales_assess = assess_trial_metric(
        measureOverTime, ts, control_store, "totSales",
        "Total sales", f"Trial assessment: Total sales (Trial {ts} vs Control {control_store})"
    )
    # Run trial assessment for customers.
    cust_assess = assess_trial_metric(
        measureOverTime, ts, control_store, "nCustomers",
        "Total customers", f"Trial assessment: Customers (Trial {ts} vs Control {control_store})"
    )

    # Build a driver summary table for this trial-control pair.
    drivers = driver_summary(measureOverTime, ts, control_store)

    # Store all outputs in a dictionary for later printing and PDF export.
    all_results.append({
        "trial_store": ts,
        "control_store": control_store,
        "score_table": score_table,
        "fig_pre_sales": fig_pre_sales,
        "fig_pre_cust": fig_pre_cust,
        "sales_assess": sales_assess,
        "cust_assess": cust_assess,
        "drivers": drivers,
    })

# GOAL: Print a quick console summary so you can sanity check results before the PDF export.
# Loop through each trial store result.
for res in all_results:
    # Print trial and control store ids.
    print(f"\nTrial store {res['trial_store']} selected control store {res['control_store']}")
    # Print sales critical t value and degrees of freedom.
    print(f"Sales t critical 95%: {res['sales_assess']['t_critical_95']:.3f} (df={res['sales_assess']['degreesOfFreedom']})")
    # Print customers critical t value and degrees of freedom.
    print(f"Customers t critical 95%: {res['cust_assess']['t_critical_95']:.3f} (df={res['cust_assess']['degreesOfFreedom']})")
    # Show top 5 control candidates.
    print("Top 5 control candidates:")
    print(res["score_table"][["Store2", "finalControlScore"]].head(5))



# GOAL: Export a PDF report with the same structure as the R template outputs (plots plus supporting tables).
# Set the output pdf file name.
OUT_PDF = "Quantium_Task2_Python_Report.pdf"
# Open a PDF writer to save multiple pages.
with PdfPages(OUT_PDF) as pdf:
    # Loop through each trial store result to add a section to the PDF.
    for res in all_results:
        # Create a summary page figure.
        fig = plt.figure(figsize=(8.5, 11))
        # Turn off axes so the page looks like a report.
        plt.axis("off")

        # Build a title line for the report page.
        title_text = f"Quantium Task 2 (Python)\nTrial store {res['trial_store']} vs Control store {res['control_store']}"
        # Place the title at the top.
        plt.text(0.5, 0.94, title_text, ha="center", va="center", fontsize=16)

        # Extract sales stats to show on the summary page.
        s_std = res["sales_assess"]["stdDev"]
        # Extract sales df to show on the summary page.
        s_df = res["sales_assess"]["degreesOfFreedom"]
        # Extract sales t critical to show on the summary page.
        s_tcrit = res["sales_assess"]["t_critical_95"]

        # Extract customer stats to show on the summary page.
        c_std = res["cust_assess"]["stdDev"]
        # Extract customer df to show on the summary page.
        c_df = res["cust_assess"]["degreesOfFreedom"]
        # Extract customer t critical to show on the summary page.
        c_tcrit = res["cust_assess"]["t_critical_95"]

        # Create a text block summarizing the sales test.
        sales_block = f"Sales assessment\nstdDev (pre-trial pct diffs): {s_std:.4f}\ndf: {s_df}\nt critical (95%): {s_tcrit:.3f}"
        # Place the sales block on the page.
        plt.text(0.1, 0.80, sales_block, ha="left", va="top", fontsize=11)

        # Create a text block summarizing the customer test.
        cust_block = f"Customer assessment\nstdDev (pre-trial pct diffs): {c_std:.4f}\ndf: {c_df}\nt critical (95%): {c_tcrit:.3f}"
        # Place the customer block on the page.
        plt.text(0.1, 0.64, cust_block, ha="left", va="top", fontsize=11)

        # Add a note about the trial window.
        plt.text(0.1, 0.52, "Trial period highlighted: Feb 2019 to Apr 2019 (201902 to 201904)", ha="left", va="top", fontsize=10)

        # Save summary page to PDF.
        pdf.savefig(fig)
        # Close the figure to free memory.
        plt.close(fig)

        # Save pre-trial sales trend plot to PDF.
        pdf.savefig(res["fig_pre_sales"])
        # Close the pre-trial sales figure.
        plt.close(res["fig_pre_sales"])

        # Save pre-trial customers trend plot to PDF.
        pdf.savefig(res["fig_pre_cust"])
        # Close the pre-trial customers figure.
        plt.close(res["fig_pre_cust"])

        # Save trial sales assessment plot to PDF.
        pdf.savefig(res["sales_assess"]["figure"])
        # Close the trial sales figure.
        plt.close(res["sales_assess"]["figure"])

        # Save trial customers assessment plot to PDF.
        pdf.savefig(res["cust_assess"]["figure"])
        # Close the trial customers figure.
        plt.close(res["cust_assess"]["figure"])

        # Create a page showing the top control store rankings.
        fig = plt.figure(figsize=(8.5, 11))
        # Turn off axes for report style.
        plt.axis("off")
        # Add a heading for the rankings table.
        plt.text(0.5, 0.95, f"Top control store rankings (Trial {res['trial_store']})", ha="center", va="center", fontsize=14)
        # Take the top 10 ranking rows.
        top10 = res["score_table"].head(10).copy()
        # Round the score columns for readability.
        top10["scoreNSales"] = top10["scoreNSales"].astype(float).round(4)
        # Round the customer score column.
        top10["scoreNCust"] = top10["scoreNCust"].astype(float).round(4)
        # Round the final score column.
        top10["finalControlScore"] = top10["finalControlScore"].astype(float).round(4)
        # Convert the top10 table to a string for PDF rendering.
        top10_text = top10[["Store2", "scoreNSales", "scoreNCust", "finalControlScore"]].to_string(index=False)
        # Print the table string onto the page using monospace font.
        plt.text(0.05, 0.90, top10_text, ha="left", va="top", family="monospace", fontsize=9)
        # Save the ranking page to PDF.
        pdf.savefig(fig)
        # Close the figure.
        plt.close(fig)

        # Create a page showing driver summary for customers vs transactions per customer etc.
        fig = plt.figure(figsize=(8.5, 11))
        # Turn off axes for report style.
        plt.axis("off")
        # Add a heading for driver summary.
        plt.text(0.5, 0.95, f"Driver summary (Trial {res['trial_store']} vs Control {res['control_store']})", ha="center", va="center", fontsize=14)
        # Round driver table for readability.
        drv = res["drivers"].copy()
        # Round numeric columns.
        for col in ["trial_pre_avg", "trial_trial_avg", "control_pre_avg", "control_trial_avg", "trial_change", "control_change", "trial_minus_control_change"]:
            drv[col] = drv[col].astype(float).round(4)
        # Convert the driver summary table to text.
        drv_text = drv.to_string(index=False)
        # Print the driver table text.
        plt.text(0.05, 0.90, drv_text, ha="left", va="top", family="monospace", fontsize=8)
        # Save the driver page to PDF.
        pdf.savefig(fig)
        # Close the figure.
        plt.close(fig)

# GOAL: Confirm the PDF report was saved.
# Print the output PDF file name.
print(f"\nSaved PDF report to: {OUT_PDF}")

# Save PNGs from figures already created in all_results
for res in all_results:
    ts = res["trial_store"]
    cs = res["control_store"]

    # Pre-trial sales chart
    res["fig_pre_sales"].savefig(
        f"pre_trial_total_sales_trial{ts}_vs_control{cs}.png",
        dpi=300,
        bbox_inches="tight"
    )

    # Trial assessment sales chart
    res["sales_assess"]["figure"].savefig(
        f"trial_assessment_total_sales_trial{ts}_vs_control{cs}.png",
        dpi=300,
        bbox_inches="tight"
    )

print("Saved PNG files for all trial stores.")
