# NMIT-SE-PROJECT-
Software Engineering projects 
# Student Management System
# Simple console-based program

import json
import os

FILE_NAME = "students.json"

def load_data():
    if os.path.exists(FILE_NAME):
        with open(FILE_NAME, "r") as file:
            return json.load(file)
    return {}

def save_data(data):
    with open(FILE_NAME, "w") as file:
        json.dump(data, file, indent=4)

def add_student(data):
    roll = input("Enter Roll Number: ")
    if roll in data:
        print("Student already exists!")
        return
    name = input("Enter Name: ")
    age = input("Enter Age: ")
    course = input("Enter Course: ")
    data[roll] = {"name": name, "age": age, "course": course}
    save_data(data)
    print("✅ Student added successfully!")

def view_students(data):
    if not data:
        print("No student records found.")
        return
    print("\n--- Student Records ---")
    for roll, info in data.items():
        print(f"Roll: {roll}, Name: {info['name']}, Age: {info['age']}, Course: {info['course']}")

def update_student(data):
    roll = input("Enter Roll Number to update: ")
    if roll not in data:
        print("Student not found.")
        return
    print("Enter new details (press enter to skip):")
    name = input(f"Name [{data[roll]['name']}]: ") or data[roll]['name']
    age = input(f"Age [{data[roll]['age']}]: ") or data[roll]['age']
    course = input(f"Course [{data[roll]['course']}]: ") or data[roll]['course']
    data[roll] = {"name": name, "age": age, "course": course}
    save_data(data)
    print("✅ Student updated successfully!")

def delete_student(data):
    roll = input("Enter Roll Number to delete: ")
    if roll in data:
        del data[roll]
        save_data(data)
        print("🗑️ Student deleted successfully!")
    else:
        print("Student not found.")

def main():
    data = load_data()
    while True:
        print("\n--- Student Management System ---")
        print("1. Add Student")
        print("2. View Students")
        print("3. Update Student")
        print("4. Delete Student")
        print("5. Exit")

        choice = input("Enter choice: ")
        if choice == "1":
            add_student(data)
        elif choice == "2":
            view_students(data)
        elif choice == "3":
            update_student(data)
        elif choice == "4":
            delete_student(data)
        elif choice == "5":
            print("Goodbye!")
            break
        else:
            print("Invalid choice. Try again.")

if __name__ == "__main__":
    main()
