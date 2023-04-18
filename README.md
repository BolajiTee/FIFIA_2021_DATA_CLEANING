INTRODUCTION

The purpose of this project is to clean a dataset of FIFA player data and prepare it for analysis. The dataset contains information on 18,207 players, including their personal details, physical attributes, performance statistics, and market values. 

The dataset was obtained from the Kaggle website and is provided in a CSV format. The original dataset contains 104 columns, but for the purpose of this project, we will focus on 77 columns that require cleaning.

Data Cleaning Steps:

To clean the data, we will first load all necessary libraries that could be used along the journey of cleaning the data. We will then load our datasets and name it “fifa_dirty”. 

The next step is to get some general information about the data and check what columns needed to be cleaned up. The dataset, from the brief information given above, shows that we have 77 columns to work with. 

This is a large number, and our Jupyter notebook cannot give us a clear picture of every column at a glance. Because we want to familiarize ourselves with every column’s data, we would divide our data along the columns to four datasets and check for missing values and know those with special characters that need to be dropped.

After checking all our columns, we will extract the columns that need cleaning into a single dataset and start working on them from column to column.