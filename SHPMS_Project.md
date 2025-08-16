# Smart Hospital Patient Management System (SHPMS)

---

## Table of Contents
1. [Database & Table Creation Scripts](#database--table-creation-scripts)
2. [Insert Sample Data](#insert-sample-data)
3. [Fabric Functions (Python)](#fabric-functions-python)
   - Add Patient
   - Schedule Appointment
   - Delete Appointment
   - Mark Bill as Paid
   - Add Patient with First Appointment
   - Add Doctor
   - Add Treatment
4. [Sample JSON Calls](#sample-json-calls)

---

## Database & Table Creation Scripts

```sql
-- Create required database and main tables for SHPMS

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

## Insert Sample Data

```sql
-- Demonstration sample records for each core entity

INSERT INTO Patients (FirstName, LastName, DOB, Gender, ContactNumber)
VALUES ('John', 'Doe', '1985-05-15', 'M', '9876543210');

INSERT INTO Doctors (FirstName, LastName, Specialization, ContactNumber)
VALUES ('Alice', 'Smith', 'Cardiology', '9876501234');

INSERT INTO Treatments (TreatmentName, Cost)
VALUES ('General Checkup', 500.00);
```

---

## Fabric Functions (Python)

### 1. Add Patient

```python
# Adds a new patient record to the Patients table.
import fabric.functions as fn
udf = fn.UserDataFunctions()

@udf.connection(argName="sqlDB", alias="SHPMSDB") 
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

### 2. Schedule Appointment

```python
# Schedules a new appointment for a patient with a doctor.
import fabric.functions as fn
udf = fn.UserDataFunctions()

@udf.connection(argName="sqlDB", alias="SHPMSDB") 
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

### 3. Delete Appointment

```python
# Removes an appointment from the schedule based on AppointmentID.
import fabric.functions as fn
udf = fn.UserDataFunctions()

@udf.connection(argName="sqlDB", alias="SHPMSDB")
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

### 4. Mark Bill as Paid

```python
# Updates a bill status to 'Paid' in the Billing table.
import fabric.functions as fn
udf = fn.UserDataFunctions()

@udf.connection(argName="sqlDB", alias="SHPMSDB")
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

### 5. Add Patient with First Appointment

```python
# Adds a new patient and immediately schedules their first appointment.
import fabric.functions as fn
udf = fn.UserDataFunctions()

@udf.connection(argName="sqlDB", alias="SHPMSDB")
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

### 6. Add Doctor

```python
# Adds a new doctor to the Doctors table.
import fabric.functions as fn
udf = fn.UserDataFunctions()

@udf.connection(argName="sqlDB", alias="SHPMSDB")
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

### 7. Add Treatment

```python
# Adds a new treatment type and its cost to the Treatments table.
import fabric.functions as fn
udf = fn.UserDataFunctions()

@udf.connection(argName="sqlDB", alias="SHPMSDB")
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

## Sample JSON Calls

```json
// Add Patient
{
  "firstName": "Jane",
  "lastName": "Doe",
  "dob": "1990-02-15",
  "gender": "F",
  "contactNumber": "9998887777"
}
```

```json
// Schedule Appointment
{
  "patientID": 1,
  "doctorID": 1,
  "appointmentDate": "2025-08-20",
  "appointmentTime": "10:30:00",
  "status": "Scheduled"
}
```

```json
// Delete Appointment
{
  "appointmentID": 1
}
```

```json
// Mark Bill Paid
{
  "billID": 1
}
```
