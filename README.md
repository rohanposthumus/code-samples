# Code samples

Due to the proprietary nature of the algorithms and software I develop, the Github repositories where they are stored are protected by non-disclosure contracts and are therefore private. However, I have created a separate Github repository specifically for code snippets that I am authorized to share or are part of my hobby projects.

## Use Python to submit a SQL query to a database
```
def download_students() -> pd.DataFrame:
    "Pull student data from server to compare with enrollments"
    with open('all_student_enrollment.sql', 'r', encoding="utf-8") as sql_script:
        sql = sql_script.read()

    pyodbc_driver = "DRIVER={SQL Server};"
    pyodbc_server = "SERVER=anonymized;"
    pyodbc_database = "DATABASE=anonymized;"

    conn = pyodbc.connect("{}{}{}".format(
        pyodbc_driver, pyodbc_server, pyodbc_database))

    df = pd.read_sql(sql, conn)
    return df
```

## Out of sample prediction with Scikit-Learn
```
import pandas as pd
import os
from datetime import datetime
from sklearn.preprocessing import StandardScaler
from sklearn.ensemble import GradientBoostingClassifier
from datetime import datetime

def predict_out_sample():
    "Predicts out of sample values using training data"
    start_time = datetime.now()
    current_time = start_time.strftime("%H:%M")

    print("[PREDICT-OUT] calling predict_in_sample() at " + current_time)

    train_dir = r"C:\anonymized\train-data"
    new_dir = r"C:\anonymized\new-data"
    training_file = "Training plus dummies.csv"
    prediction_file = "No results.csv"
    updating_file = "prediction results master list.csv"
    student_number_str = "student_number"
    predict_variable = "score_category"

    print("[PREDICT-OUT] Importing data")

    os.chdir(train_dir)
    training_data = pd.read_csv(training_file,
                                encoding='windows-1254',
                                engine='python',
                                dtype={student_number_str: str})
    training_data.drop([student_number_str,
                        'campus',
                        'faculty'], axis=1, inplace=True)

    training_columns = training_data.iloc[0:0]

    os.chdir(new_dir)
    prediction_data = pd.read_csv(
        prediction_file,
        encoding='windows-1254',
        engine='python',
        dtype={student_number_str: str})
    prediction_data = training_columns.append(prediction_data)
    prediction_data = prediction_data.fillna(0)

    student_numbers = prediction_data[student_number_str].copy() # Use later
    prediction_data.drop([student_number_str,
                          'score',  
                          'campus',
                          'faculty'],
                         axis=1, inplace=True)

    prediction_data[predict_variable] = "no_results_yet"

    print("[PREDICT-OUT] Splitting data for testing")

    X_train = (training_data.drop(
        [predict_variable], axis=1)).values      # features
    Y_train = (training_data[predict_variable]
               ).values                          # target

    X_validation = (prediction_data.drop(
        [predict_variable], axis=1)).values      

    scaler = StandardScaler().fit(X_train)
    X_train = scaler.transform(X_train)
    X_validation = scaler.transform(X_validation)

    print("[PREDICT-OUT] Prediction")

    seed = 7
    model = GradientBoostingClassifier(random_state=seed)
    model.fit(X_train, Y_train)
    Y_model_predict = model.predict(X_validation)

    prediction_data[predict_variable] = Y_model_predict

    print("[PREDICT-OUT] Saving")
    
    prediction_data[student_number_str] = student_numbers
    prediction_data["date_of_analysis"] = str(
        datetime.now().strftime("%m/%d/%Y, %H:%M:%S"))
    prediction_data = prediction_data[[student_number_str,
                                       "score_category",
                                       "date_of_analysis"]]
    prediction_data.to_csv(
        "prediction results.csv", index=False)

    updating_data = pd.read_csv(updating_file,
                                encoding='windows-1254',
                                engine='python',
                                dtype={student_number_str: str})

    refresh_updated_data = updating_data.append(prediction_data)
    refresh_updated_data.sort_values(
        by=["date_of_analysis"], ascending=False, inplace=True)
    refresh_updated_data.drop_duplicates(
        subset=[student_number_str], inplace=True)
    refresh_updated_data.to_csv(
        "prediction results master list.csv", index=False)

    print("[PREDICT-OUT] calling predict() at " + current_time)

    finish_time = (datetime.now() - start_time).total_seconds()
    if finish_time < 60:
        print("[PREDICT-OUT] predict() finished",
              round(finish_time), "seconds later")
    else:
        print("[PREDICT-OUT] predict() finished",
              round(finish_time/60), "minutes later")
```
