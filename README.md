背景:病房管理是系统医院不可或缺的系统,对于医院的正常运转至关重要
数据库:mySQL
编程语言:python(pymysql库)
可视化客户端:tkinter

在该系统中,病人有九个变量如下:

    create_table_query = """
        CREATE TABLE IF NOT EXISTS patients (
            id INT AUTO_INCREMENT PRIMARY KEY,
            ward_number VARCHAR(10),
            name VARCHAR(100),
            id_number VARCHAR(20) UNIQUE,
            gender ENUM('Male', 'Female', 'Other'),
            age INT,
            phone_number VARCHAR(15),
            attending_doctor VARCHAR(100),
            blood_type ENUM('A', 'B', 'AB', 'O', 'Unknown')
        )
        """
病人每个病房设置为有16个床位

![image.png][1]
按钮1:添加病人
![image.png][2]
按钮2:查询病人
![image.png][3]
按钮3:修改病人
![image.png][4]
按钮4:查看病房情况
最上方显示一共有多少个病房以及一共有多少个空的床位
并且显示每一个病房以及容量情况,如果容量满则显示红色的Full,否则显示绿色的Free
![image.png][5]


  [1]: http://8.217.166.104/usr/uploads/2024/10/2791800278.png
  [2]: http://8.217.166.104/usr/uploads/2024/10/1121422865.png
  [3]: http://8.217.166.104/usr/uploads/2024/10/2638841655.png
  [4]: http://8.217.166.104/usr/uploads/2024/10/3064605184.png
  [5]: http://8.217.166.104/usr/uploads/2024/10/2378232421.png


源代码:

    import tkinter as tk
from tkinter import ttk, messagebox
import pymysql
from typing import List, Tuple, Optional

class Database:
    def __init__(self):
        self.connection = pymysql.connect(
            host='localhost',
            user='root',  # replace with your MySQL username
            password='123456',  # replace with your MySQL password
            database='medical_ward'
        )
        self.cursor = self.connection.cursor()
        self.create_table()

    def create_table(self):
        create_table_query = """
        CREATE TABLE IF NOT EXISTS patients (
            id INT AUTO_INCREMENT PRIMARY KEY,
            ward_number VARCHAR(10),
            name VARCHAR(100),
            id_number VARCHAR(20) UNIQUE,
            gender ENUM('Male', 'Female', 'Other'),
            age INT,
            phone_number VARCHAR(15),
            attending_doctor VARCHAR(100),
            blood_type ENUM('A', 'B', 'AB', 'O', 'Unknown')
        )
        """
        self.cursor.execute(create_table_query)
        self.connection.commit()

    def add_patient(self, patient_data: dict) -> bool:
        add_query = """
        INSERT INTO patients 
        (ward_number, name, id_number, gender, age, phone_number, attending_doctor, blood_type) 
        VALUES (%(ward_number)s, %(name)s, %(id_number)s, %(gender)s, %(age)s, %(phone_number)s, %(attending_doctor)s, %(blood_type)s)
        """
        try:
            self.cursor.execute(add_query, patient_data)
            self.connection.commit()
            return True
        except pymysql.Error as e:
            print(f"Error adding patient: {e}")
            return False

    def query_patient(self, query_type: str, value: str) -> Optional[Tuple]:
        query = f"SELECT * FROM patients WHERE {query_type} = %s"
        self.cursor.execute(query, (value,))
        return self.cursor.fetchone()

    def update_patient(self, patient_id: int, patient_data: dict) -> bool:
        update_query = """
        UPDATE patients SET
        ward_number = %(ward_number)s,
        name = %(name)s,
        id_number = %(id_number)s,
        gender = %(gender)s,
        age = %(age)s,
        phone_number = %(phone_number)s,
        attending_doctor = %(attending_doctor)s,
        blood_type = %(blood_type)s
        WHERE id = %(id)s
        """
        patient_data['id'] = patient_id
        try:
            self.cursor.execute(update_query, patient_data)
            self.connection.commit()
            return True
        except pymysql.Error as e:
            print(f"Error updating patient: {e}")
            return False

    def get_ward_stats(self) -> Tuple[int, int, List[Tuple[str, int, int]]]:
        self.cursor.execute("SELECT COUNT(DISTINCT ward_number) FROM patients")
        total_wards = self.cursor.fetchone()[0]

        self.cursor.execute("SELECT ward_number, COUNT(*) FROM patients GROUP BY ward_number")
        ward_occupancy = self.cursor.fetchall()

        ward_capacity = {ward: 16 for ward, _ in ward_occupancy}  # Assuming each ward has a capacity of 16

        total_capacity = sum(ward_capacity.values())
        occupied_beds = sum(count for _, count in ward_occupancy)
        vacant_beds = total_capacity - occupied_beds

        ward_stats = [(ward, count, ward_capacity[ward]) for ward, count in ward_occupancy]
        ward_stats.sort(key=lambda x: x[0])  # Sort wards in ascending order of their numbers

        return total_wards, vacant_beds, ward_stats

class WardManagementSystem:
    def __init__(self, master):
        self.master = master
        self.master.title("Ward Management System")
        self.master.geometry("800x600")  # Increase the window height
        self.db = Database()

        self.create_main_menu()

    def create_main_menu(self):
        self.clear_window()

        ttk.Button(self.master, text="Add Patient", command=self.add_patient_window).pack(pady=10)
        ttk.Button(self.master, text="Query Patient", command=self.query_patient_window).pack(pady=10)
        ttk.Button(self.master, text="Modify Patient", command=self.modify_patient_window).pack(pady=10)
        ttk.Button(self.master, text="View Ward Situation", command=self.view_ward_situation).pack(pady=10)

    def clear_window(self):
        for widget in self.master.winfo_children():
            widget.destroy()

    def add_patient_window(self):
        self.clear_window()

        fields = ['Ward Number', 'Name', 'ID Number', 'Gender', 'Age', 'Phone Number', 'Attending Doctor', 'Blood Type']
        entries = {}

        for field in fields:
            ttk.Label(self.master, text=field).pack()
            entry = ttk.Entry(self.master)
            entry.pack()
            entries[field.lower().replace(' ', '_')] = entry

        ttk.Button(self.master, text="Submit", command=lambda: self.submit_patient(entries)).pack(pady=10)
        ttk.Button(self.master, text="Back", command=self.create_main_menu).pack(pady=10)

    def submit_patient(self, entries):
        patient_data = {key: entry.get() for key, entry in entries.items()}
        if all(patient_data.values()):
            if self.db.add_patient(patient_data):
                messagebox.showinfo("Success", "Patient added successfully")
                self.create_main_menu()
            else:
                messagebox.showerror("Error", "Failed to add patient")
        else:
            messagebox.showerror("Error", "All fields must be filled")

    def query_patient_window(self):
        self.clear_window()

        ttk.Label(self.master, text="Query by:").pack()
        query_type = tk.StringVar()
        ttk.Radiobutton(self.master, text="Name", variable=query_type, value="name").pack()
        ttk.Radiobutton(self.master, text="ID Number", variable=query_type, value="id_number").pack()

        ttk.Label(self.master, text="Enter value:").pack()
        query_value = ttk.Entry(self.master)
        query_value.pack()

        ttk.Button(self.master, text="Query", command=lambda: self.perform_query(query_type.get(), query_value.get())).pack(pady=10)
        ttk.Button(self.master, text="Back", command=self.create_main_menu).pack(pady=10)

    def perform_query(self, query_type, value):
        result = self.db.query_patient(query_type, value)
        if result:
            self.display_patient_info(result)
        else:
            messagebox.showinfo("Result", "No patient found")

    def display_patient_info(self, patient_info):
        self.clear_window()

        fields = ['ID', 'Ward Number', 'Name', 'ID Number', 'Gender', 'Age', 'Phone Number', 'Attending Doctor', 'Blood Type']

        frame = ttk.Frame(self.master)
        frame.pack(fill=tk.BOTH, expand=True)

        canvas = tk.Canvas(frame)
        canvas.pack(side=tk.LEFT, fill=tk.BOTH, expand=True)

        scrollbar = ttk.Scrollbar(frame, orient=tk.VERTICAL, command=canvas.yview)
        scrollbar.pack(side=tk.RIGHT, fill=tk.Y)

        canvas.configure(yscrollcommand=scrollbar.set)
        canvas.bind('<Configure>', lambda e: canvas.configure(scrollregion=canvas.bbox('all')))

        inner_frame = ttk.Frame(canvas)
        canvas.create_window((0, 0), window=inner_frame, anchor='nw')

        for field, value in zip(fields, patient_info):
            ttk.Label(inner_frame, text=f"{field}: {value}").pack()

        ttk.Button(inner_frame, text="Back", command=self.query_patient_window).pack(pady=10)
        ttk.Button(inner_frame, text="Modify", command=lambda: self.update_patient_info(patient_info[0])).pack(pady=10)

    def modify_patient_window(self):
        self.query_patient_window()

    def update_patient_info(self, patient_id):
        patient_info = self.db.query_patient("id", patient_id)
        if patient_info:
            self.clear_window()

            fields = ['Ward Number', 'Name', 'ID Number', 'Gender', 'Age', 'Phone Number', 'Attending Doctor', 'Blood Type']
            entries = {}

            for field, value in zip(fields, patient_info[1:]):
                ttk.Label(self.master, text=field).pack()
                entry = ttk.Entry(self.master)
                entry.insert(0, value)
                entry.pack()
                entries[field.lower().replace(' ', '_')] = entry

            ttk.Button(self.master, text="Update", command=lambda: self.submit_update(patient_id, entries)).pack(pady=10)
            ttk.Button(self.master, text="Back", command=self.query_patient_window).pack(pady=10)
        else:
            messagebox.showerror("Error", "Patient not found")

    def submit_update(self, patient_id, entries):
        patient_data = {key: entry.get() for key, entry in entries.items()}
        if all(patient_data.values()):
            if self.db.update_patient(patient_id, patient_data):
                messagebox.showinfo("Success", "Patient updated successfully")
                self.query_patient_window()
            else:
                messagebox.showerror("Error", "Failed to update patient")
        else:
            messagebox.showerror("Error", "All fields must be filled")

    def view_ward_situation(self):
        self.clear_window()

        total_wards, vacant_beds, ward_stats = self.db.get_ward_stats()

        ttk.Label(self.master, text=f"Currently, there are {total_wards} wards and {vacant_beds} vacant beds").pack(pady=10)

        frame = ttk.Frame(self.master)
        frame.pack(fill=tk.BOTH, expand=True)

        canvas = tk.Canvas(frame)
        canvas.pack(side=tk.LEFT, fill=tk.BOTH, expand=True)

        scrollbar = ttk.Scrollbar(frame, orient=tk.VERTICAL, command=canvas.yview)
        scrollbar.pack(side=tk.RIGHT, fill=tk.Y)

        canvas.configure(yscrollcommand=scrollbar.set)
        canvas.bind('<Configure>', lambda e: canvas.configure(scrollregion=canvas.bbox('all')))

        inner_frame = ttk.Frame(canvas)
        canvas.create_window((0, 0), window=inner_frame, anchor='nw')

        for ward, current_capacity, total_capacity in ward_stats:
            status = "Free" if current_capacity < total_capacity else "Full"
            color = "green" if current_capacity < total_capacity else "red"
            ttk.Label(inner_frame, text=f"Ward {ward}: {current_capacity}/{total_capacity} ", foreground=color).pack()
            ttk.Label(inner_frame, text=status, foreground=color).pack()

        ttk.Button(inner_frame, text="Back", command=self.create_main_menu).pack(pady=10)

if __name__ == "__main__":
    root = tk.Tk()
    app = WardManagementSystem(root)
    root.mainloop()
