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

## Machine learning prediction with Scikit-Learn
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

    print("[PREDICT-OUT] Calling predict_out_sample() at " + current_time)

    train_dir = r"C:\anonymized\train-data"
    new_dir = r"C:\anonymized\new-data"
    training_file = "training plus dummies.csv"
    prediction_file = "no results.csv"
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

    # Will use student numbers later
    student_numbers = prediction_data[student_number_str].copy() 
    prediction_data.drop([student_number_str,
                          'score',  
                          'campus',
                          'faculty'], axis=1, inplace=True)

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

    finish_time = (datetime.now() - start_time).total_seconds()
    if finish_time < 60:
        print("[PREDICT-OUT] predict_out_sample() finished",
              round(finish_time), "seconds later")
    else:
        print("[PREDICT-OUT] predict_out_sample() finished",
              round(finish_time/60), "minutes later")
```
## Simple SQL query
```
DECLARE @StartDate AS VARCHAR(100) = '2022-07-01';

SELECT json_value(lp.stage, '$.user_id') AS StudentNumber,
    lt.name AS ToolName,
    lci.name AS CourseItemName,
    lci.item_type AS ItemType,
    lc.name AS CourseName,
    lc.course_number AS CourseNumber,
    COUNT(DISTINCT lcia.person_id) AS DistinctItemUsers,
    lcia.duration_sum / 60 AS DurationMinutes,
    lcia.interaction_cnt AS CourseItemClicks,
    convert(VARCHAR, lcia.first_accessed_time, 103) AS 'Date'
FROM stg_lms.cdm_lms.course AS lc
LEFT JOIN stg_lms.cdm_lms.course_item AS lci ON lc.id = lci.course_id
LEFT JOIN stg_lms.cdm_lms.course_tool AS lct ON lci.course_tool_id = lct.id
LEFT JOIN stg_lms.cdm_lms.tool AS lt ON lt.id = lct.tool_id
LEFT JOIN stg_lms.cdm_lms.course_item_activity AS lcia ON lcia.course_item_id = lci.id
LEFT JOIN stg_lms.cdm_lms.person_course AS lpc ON lpc.id = lcia.person_course_id
LEFT JOIN stg_lms.cdm_lms.person AS lp ON lpc.person_id = lp.id
WHERE lpc.course_role = 'S'
    AND lc.course_number IN ('anonymized')
    AND lcia.first_accessed_time >= @StartDate
GROUP BY lci.item_type,
    lt.tool_type,
    lt.name,
    lci.name,
    lc.name,
    lc.course_number,
    lcia.first_accessed_time,
    json_value(lp.stage, '$.user_id'),
    lcia.duration_sum,
    lcia.interaction_cnt
```
## Use a dataframe to send emails using HTML templates
```
import pandas as pd
import os
from datetime import datetime
import win32com.client as win32
from jinja2 import Template
import letters.py_2023.template_bf as template1
import letters.py_2023.template_qq as template2

outlook = win32.Dispatch('outlook.application')

def run(df_email_preprocess: pd.DataFrame, mode: str = "display"):
    "Use a dataframe to send emails using HTML templates."
    try:
        start_time = datetime.now()

        # Confirm with user before sending thousands of emails
        print("Do you want to run the script? (y/n)")
        x = input()

        if x != "y":
            print("Aborting email bot")
            raise SystemExit

        # If the mode is incorrect, exit
        if mode != "display" and mode != "send":
            print("Aborting email bot")
            raise SystemExit

        # Get the files for attachments
        start_dir = os.getcwd()

        data1 = df_email_preprocess
        data1 = data1.drop_duplicates(subset=["student_number"])

        # Get the images for attachments
        attachment_path = start_dir + '\\letters\\py_2023'

        attachment1_source = os.path.join(
            attachment_path, "Top_anonymized.png")
        attachment2_source = os.path.join(
            attachment_path, "Bottom_anonymized.png")
        property_accessor = r"http://schemas.microsoft.com/mapi/proptag/0x3712001F"

        print('[STATUS] Looping through email numbers')
        for dec, num, nam, ema in zip(data1["decision"], data1["student_number"], data1["first_name"], data1["email"]):
            # Switch email accounts based on decision and use different template each time
            if dec == "anonymized1":
                mail = outlook.CreateItem(0)
                mail.SentOnBehalfOfName = "anonymized1@anonymized.com"
                mail.To = str(ema)
                mail.Subject = '{}: anonymized message'.format(
                    num)
                attachment1 = mail.Attachments.Add(attachment1_source)
                attachment1.PropertyAccessor.SetProperty(
                    property_accessor, "Attachment-Header")
                attachment2 = mail.Attachments.Add(attachment2_source)
                attachment2.PropertyAccessor.SetProperty(
                    property_accessor, "Attachment-Footer")
                mail.HTMLBody = template1.bf.render(
                    student_name=nam, student_number=num)
                if mode == "display":
                    mail.display()  # This only writes the email but does not send it
                elif mode == "send":
                    mail.Send()     # Send emails i.e. live
                print("Template1 email send to {}: {}".format(nam, num))

            # Code omitted for brevity

            elif dec == "anonymized2":
                mail = outlook.CreateItem(0)
                mail.SentOnBehalfOfName = "anonymized2@anonymized.com"
                mail.To = str(ema)
                mail.Subject = '{}: anonymized message'.format(
                    num)
                attachment1 = mail.Attachments.Add(attachment1_source)
                attachment1.PropertyAccessor.SetProperty(
                    property_accessor, "Attachment-Header")
                attachment2 = mail.Attachments.Add(attachment2_source)
                attachment2.PropertyAccessor.SetProperty(
                    property_accessor, "Attachment-Footer")
                mail.HTMLBody = template2.qq.render(
                    student_name=nam, student_number=num)
                if mode == "display":
                    mail.display()
                elif mode == "send":
                    mail.Send()    
                print("Template2 email send to {}: {}".format(nam, num))

            # Code omitted for brevity

            else:
                print("No match {}: {}".format(nam, num))

    except Exception as e:
        print(e)
```

## Dataframes and recursion
```
# Code omitted for brevity
sys.setrecursionlimit(1000)

def algo_manage_dependencies(df: pd.DataFrame, card_id: str, dependency: str, priority_number: str, row: int = 0, viewed_cards=[], count=0) -> pd.DataFrame:
    "This function makes sure that the order of dependencies does not clash."

    try:
        df.sort_values(by=[priority_number],
                       inplace=True,
                       ascending=[True])
        viewed_cards = []
        score_list = []

        for w, d in zip(df[card_id], df[dependency]):
            try:
                if d == '0':
                    score_list.append(1)
                    viewed_cards.append(w)

                elif d in viewed_cards:
                    score_list.append(1)
                    viewed_cards.append(w)
                else:
                    score_list.append(0)
                    viewed_cards.append(w)
            except:
                score_list.append(0)
                viewed_cards.append(w)

        df['score'] = score_list
        sum_of_deps = df['score'].sum(axis=0)
        n = len(df)
        j = row
        k = j + 1

        if sum_of_deps != n and j < n and k < n:
            # print(viewed_cards)
            if df['score'].iloc[j] == 0:
                df[priority_number].iloc[j] = df[priority_number].iloc[j] + 2
                count += 1
                viewed_cards = viewed_cards.remove(df[card_id].iloc[j])
                return algo_manage_dependencies(df, 'card_id', 'card_dependency', 'priority_number', j, viewed_cards, count)
            elif df['score'].iloc[j] == 1:
                count += 1
                return algo_manage_dependencies(df, 'card_id', 'card_dependency', 'priority_number', k, viewed_cards, count)
            else:
                pass

        return df
    except RecursionError as re:
        # Code omitted for brevity
        return df

    except Exception as e:
        # Code omitted for brevity
        df = pd.DataFrame()
        return df

```

## Multiprocessing
```
from multiprocessing import Process, Queue, current_process
# Code omitted for brevity

def queue_writer(sql_query: str, database: str, message_queue: Queue) -> None:
    "Read dataframe from database and put in message queue."
    name = current_process().name
    print("[PREPROCESSING] {} started".format(name))
    try:
        with open(sql_query, 'r', encoding="utf-8") as sql_script:
            sql = sql_script.read()

        pyodbc_driver = "DRIVER={SQL Server};"
        pyodbc_server = "SERVER=anonymized;"
        pyodbc_database = "DATABASE={};".format(database) # Pass database string dynamically

        conn = pyodbc.connect("{}{}{}".format(
            pyodbc_driver, pyodbc_server, pyodbc_database))

        df = pd.read_sql(sql, conn)
        message_queue.put(df)

        print("[PREPROCESSING] {} ended: Success".format(name))
    except Exception as e:
        print("[PREPROCESSING] {} ended: Failure.\n[Error] \t{}.\n[Drivers] \t{}\n".format(
            name, e, pyodbc.drivers()))


def get_data() -> pd.DataFrame:
    """Loop through list containing SQL queries and database names,
    Submit queries to database and process dataframes."""
    os.chdir('.\\db')

    message_queue = Queue()

    child_processes = []
    for i in range(len(cg.Globals.data)):
        p = Process(target=queue_writer, args=[
                    cg.Globals.data[i][0], cg.Globals.data[i][1], message_queue])
        p.start()
        child_processes.append(p)

    queue_results = []
    while True:
        try:
            r = message_queue.get(block=False, timeout=0.01)
            queue_results.append(r)
        except queue.Empty:
            pass
        all_exited = True
        for t in child_processes:
            try:
                if t.exitcode is None:
                    all_exited = False
                    break
            except:
                pass
        if all_exited & message_queue.empty():
            break

    for p in child_processes:
        p.join()

    # Now join with master file
    with open(cg.Globals.sql_query_population, 'r', encoding="utf-8") as sql_script:
        sql = sql_script.read()

    pyodbc_driver = "DRIVER={SQL Server};"
    pyodbc_server = "SERVER=anonymized;"
    pyodbc_database = "DATABASE={};".format(cg.Globals.database_sis)

    conn = pyodbc.connect("{}{}{}".format(
        pyodbc_driver, pyodbc_server, pyodbc_database))

    master_df = pd.read_sql(sql, conn)

    # Left join
    for frames in queue_results:
        master_df = master_df.merge(frames,
                                    on='StudentNumber',
                                    how='left')

    master_df.to_excel("Data extraction ran on {}.xlsx".format(
        time), sheet_name='Sheet1', index=False)
    return master_df

```