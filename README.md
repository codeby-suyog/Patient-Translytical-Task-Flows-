# Smart Hospital Patient Management System (SHPMS)

## 1. Introduction
The **Smart Hospital Patient Management System (SHPMS)** is a database and function-based system designed to manage patients, doctors, appointments, treatments, and billing within a healthcare environment. This project integrates with Microsoft Fabric to enable real-time operations via Fabric Functions and Power BI.

---

## 2. Project Setup Instructions
### Prerequisites
- Microsoft Fabric workspace
- SQL Endpoint enabled
- Python environment with `fabric` module
- Power BI (optional for reporting)

### Steps
1. Run the SQL scripts in section **3** to create the database and tables.
2. Deploy the Python Fabric Functions in section **5** into your Fabric environment.
3. Test using the JSON request bodies from section **6**.

---

## 3. Database & Table Creation Scripts

```sql
CREATE DATABASE SHPMS_DB;
GO

USE SHPMS_DB;
GO

CREATE TABLE Patients (
    PatientID INT PRIMARY KEY IDENTITY(1,1),
    FirstName NVARCHAR(50),
    LastName NVARCHAR(50),
    DOB DATE,
    Gender CHAR(1),
    ContactNumber NVARCHAR(15)
);

CREATE TABLE Doctors (
    DoctorID INT PRIMARY KEY IDENTITY(1,1),
    FirstName NVARCHAR(50),
    LastName NVARCHAR(50),
    Specialization NVARCHAR(100),
    ContactNumber NVARCHAR(15)
);

CREATE TABLE Treatments (
    TreatmentID INT PRIMARY KEY IDENTITY(1,1),
    TreatmentName NVARCHAR(100),
    Cost DECIMAL(10,2)
);

CREATE TABLE Appointments (
    AppointmentID INT PRIMARY KEY IDENTITY(1,1),
    PatientID INT FOREIGN KEY REFERENCES Patients(PatientID),
    DoctorID INT FOREIGN KEY REFERENCES Doctors(DoctorID),
    AppointmentDate DATE,
    AppointmentTime TIME,
    Status NVARCHAR(20)
);

CREATE TABLE Billing (
    BillID INT PRIMARY KEY IDENTITY(1,1),
    AppointmentID INT FOREIGN KEY REFERENCES Appointments(AppointmentID),
    Amount DECIMAL(10,2),
    PaymentStatus NVARCHAR(20)
);
```

---

## 4. Insert Sample Data

```sql
INSERT INTO Patients (FirstName, LastName, DOB, Gender, ContactNumber)
VALUES ('John', 'Doe', '1985-05-15', 'M', '9876543210');

INSERT INTO Doctors (FirstName, LastName, Specialization, ContactNumber)
VALUES ('Alice', 'Smith', 'Cardiology', '9876501234');

INSERT INTO Treatments (TreatmentName, Cost)
VALUES ('General Checkup', 500.00);
```

---

## 5. Fabric Functions

### Add Patient
```python
import fabric.functions as fn
udf = fn.UserDataFunctions()

@udf.connection(argName="sqlDB", alias="SHPMSDB") '''uses SHPMSDB as the connection alias to your SQL database. Please replace "SHPMSDB" with your own SQL database connection alias if it is different.'''
@udf.function()
def add_patient(sqlDB, firstName, lastName, dob, gender, contactNumber):
    connection = sqlDB.connect()
    cursor = connection.cursor()
    cursor.execute(
        "INSERT INTO Patients (FirstName, LastName, DOB, Gender, ContactNumber) VALUES (?, ?, ?, ?, ?)",
        (firstName, lastName, dob, gender, contactNumber)
    )
    connection.commit()
    cursor.close()
    connection.close()
    return "Patient added successfully"
```

### Schedule Appointment
```python
import fabric.functions as fn
udf = fn.UserDataFunctions()

@udf.connection(argName="sqlDB", alias="SHPMSDB") '''uses SHPMSDB as the connection alias to your SQL database. Please replace "SHPMSDB" with your own SQL database connection alias if it is different.'''
@udf.function()
def schedule_appointment(sqlDB, patientID, doctorID, appointmentDate, appointmentTime, status):
    if status not in ['Scheduled', 'Completed', 'Cancelled']:
        raise fn.UserThrownError("Invalid status value.", {"Status": status})
    connection = sqlDB.connect()
    cursor = connection.cursor()
    cursor.execute(
        "INSERT INTO Appointments (PatientID, DoctorID, AppointmentDate, AppointmentTime, Status) VALUES (?, ?, ?, ?, ?)",
        (patientID, doctorID, appointmentDate, appointmentTime, status)
    )
    connection.commit()
    cursor.close()
    connection.close()
    return "Appointment scheduled"
```

### Delete Appointment
```python
import fabric.functions as fn
udf = fn.UserDataFunctions()

@udf.connection(argName="sqlDB", alias="SHPMSDB") '''uses SHPMSDB as the connection alias to your SQL database. Please replace "SHPMSDB" with your own SQL database connection alias if it is different.'''
@udf.function()
def delete_appointment(sqlDB, appointmentID):
    connection = sqlDB.connect()
    cursor = connection.cursor()
    cursor.execute("DELETE FROM Appointments WHERE AppointmentID = ?", (appointmentID,))
    connection.commit()
    cursor.close()
    connection.close()
    return "Appointment deleted"
```

### Mark Bill as Paid
```python
import fabric.functions as fn
udf = fn.UserDataFunctions()

@udf.connection(argName="sqlDB", alias="SHPMSDB")  '''uses SHPMSDB as the connection alias to your SQL database. Please replace "SHPMSDB" with your own SQL database connection alias if it is different.'''
@udf.function()
def mark_bill_paid(sqlDB, billID):
    connection = sqlDB.connect()
    cursor = connection.cursor()
    cursor.execute("UPDATE Billing SET PaymentStatus = 'Paid' WHERE BillID = ?", (billID,))
    connection.commit()
    cursor.close()
    connection.close()
    return "Bill marked as paid"
```

### Add Patient with First Appointment
```python
import fabric.functions as fn
udf = fn.UserDataFunctions()

@udf.connection(argName="sqlDB", alias="SHPMSDB")  '''uses SHPMSDB as the connection alias to your SQL database. Please replace "SHPMSDB" with your own SQL database connection alias if it is different.'''
@udf.function()
def add_patient_with_first_appointment(sqlDB, firstName, lastName, dob, gender, contactNumber, doctorID, appointmentDate, appointmentTime, status):
    if status not in ['Scheduled', 'Completed', 'Cancelled']:
        raise fn.UserThrownError("Invalid status value.", {"Status": status})
    connection = sqlDB.connect()
    cursor = connection.cursor()
    cursor.execute(
        "INSERT INTO Patients (FirstName, LastName, DOB, Gender, ContactNumber) OUTPUT INSERTED.PatientID VALUES (?, ?, ?, ?, ?)",
        (firstName, lastName, dob, gender, contactNumber)
    )
    patientID = cursor.fetchall()[0][0]
    cursor.execute(
        "INSERT INTO Appointments (PatientID, DoctorID, AppointmentDate, AppointmentTime, Status) VALUES (?, ?, ?, ?, ?)",
        (patientID, doctorID, appointmentDate, appointmentTime, status)
    )
    connection.commit()
    cursor.close()
    connection.close()
    return "Patient and appointment added successfully"
```

### Add Doctor
```python
import fabric.functions as fn
udf = fn.UserDataFunctions()

@udf.connection(argName="sqlDB", alias="SHPMSDB")  '''uses SHPMSDB as the connection alias to your SQL database. Please replace "SHPMSDB" with your own SQL database connection alias if it is different.'''
@udf.function()
def add_doctor(sqlDB, firstName, lastName, specialization, contactNumber):
    connection = sqlDB.connect()
    cursor = connection.cursor()
    cursor.execute(
        "INSERT INTO Doctors (FirstName, LastName, Specialization, ContactNumber) VALUES (?, ?, ?, ?)",
        (firstName, lastName, specialization, contactNumber)
    )
    connection.commit()
    cursor.close()
    connection.close()
    return "Doctor added successfully"
```

### Add Treatment
```python
import fabric.functions as fn
udf = fn.UserDataFunctions()

@udf.connection(argName="sqlDB", alias="SHPMSDB")  '''uses SHPMSDB as the connection alias to your SQL database. Please replace "SHPMSDB" with your own SQL database connection alias if it is different.'''
@udf.function()
def add_treatment(sqlDB, treatmentName, cost):
    connection = sqlDB.connect()
    cursor = connection.cursor()
    cursor.execute(
        "INSERT INTO Treatments (TreatmentName, Cost) VALUES (?, ?)",
        (treatmentName, cost)
    )
    connection.commit()
    cursor.close()
    connection.close()
    return "Treatment added successfully"
```

---

## 6. Sample JSON Calls

### Add Patient
```json
{
  "firstName": "Jane",
  "lastName": "Doe",
  "dob": "1990-02-15",
  "gender": "F",
  "contactNumber": "9998887777"
}
```

### Schedule Appointment
```json
{
  "patientID": 1,
  "doctorID": 1,
  "appointmentDate": "2025-08-20",
  "appointmentTime": "10:30:00",
  "status": "Scheduled"
}
```

### Delete Appointment
```json
{
  "appointmentID": 1
}
```

### Mark Bill Paid
```json
{
  "billID": 1
}
```

---

## 7. Notes & Recommendations
- Always validate foreign key values before inserting (DoctorID, PatientID).
- Ensure proper error handling in Fabric Functions.
- Use parameterized queries to prevent SQL injection.
- Extend tables for more detailed records (e.g., address, medical history).
