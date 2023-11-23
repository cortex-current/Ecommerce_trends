# Predicting how a customer will feel about a product before they even ordered it

We want to predict the customer product satisfaction score for the next order or purchase which will help make better decisions. But it is not enough to just train the model once.

Instead, we are building an end-to-end pipeline for continuously predicting and deploying the machine learning model, alongside a data application that displays the latest deployed model for the business stakeholders.

In particular, we utilize MLflow [MLFlow](https://mlflow.org/) tracking to track our metrics and parameters, and MLflow deployment to deploy our model. We also use [Streamlit](https://streamlit.io/) to showcase how this model will be used in a real-world setting.
