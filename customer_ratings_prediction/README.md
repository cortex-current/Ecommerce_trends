# Predicting how a customer will feel about a product before they even ordered it

We want to predict the customer product satisfaction score for the next order or purchase which will help make better decisions. But it is not enough to just train the model once.

Instead, we are building an end-to-end pipeline for continuously predicting and deploying the machine learning model, alongside a data application that displays the latest deployed model for the business stakeholders.

In particular, we utilize MLflow [MLFlow](https://mlflow.org/) tracking to track our metrics and parameters, and MLflow deployment to deploy our model. We also use [Streamlit](https://streamlit.io/) to showcase how this model will be used in a real-world setting.

### Training Pipeline

Our standard training pipeline consists of several steps:

- `ingest_data`: This step will ingest the data and create a `DataFrame`.
- `clean_data`: This step will clean the data and remove the unwanted columns.
- `train_model`: This step will train the model and save the model using [MLflow autologging](https://www.mlflow.org/docs/latest/tracking.html).
- `evaluation`: This step will evaluate the model and save the metrics -- using MLflow autologging -- into the artifact store.

## Demo Streamlit App

There is a live demo of this project using [Streamlit](https://streamlit.io/) which you can find [here](https://share.streamlit.io//customer-satisfaction/main). It takes some input features for the product and predicts the customer satisfaction rate using the latest trained models. If you want to run this Streamlit app in your local system, you can run the following command:-

```bash
streamlit run streamlit_app.py
```

## Reference

I have created this end-to-end pipeline by learning from Ayush Singh's [project](https://github.com/ayush714/customer-satisfaction-mlops) but I have re-created this without using the ZenML framework for training and deployment.
