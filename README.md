import sqlite3
import tkinter as tk
from threading import Thread
from tkinter import messagebox, ttk

from tkcalendar import Calendar


# Initialize a global variable to track if the calendar is open
is_calendar_open = False


def open_calendar(date_entry=None):
    """
    Opens a calendar popup that allows users to select a date.
    Restricts the calendar to only open once.
    """
    global is_calendar_open  # Refer to the global variable

    # Check if the calendar is already open
    if is_calendar_open:
        messagebox.showinfo("Calendar Limit", "The calendar is already open!")  # Show warning
        return

    # Mark the calendar as open
    is_calendar_open = True

    def select_date():
        """
        Handles date selection. Sets the date in the input field and closes the calendar.
        """
        selected_date = cal.get_date()  # Get the selected date from the calendar
        date_entry.delete(0, tk.END)  # Delete any existing text in the entry field
        date_entry.insert(0, selected_date)  # Insert the selected date
        close_calendar()  # Close the calendar window

    def close_calendar():
        """
        Closes the calendar window and marks it as not open.
        """
        global is_calendar_open
        is_calendar_open = False  # Reset the calendar open flag
        calendar_window.destroy()  # Destroy the calendar window

    # Create a top-level window for the calendar
    calendar_window = tk.Toplevel(app)
    calendar_window.title("Select Date")

    # Handle the case where the user manually closes the calendar window
    calendar_window.protocol("WM_DELETE_WINDOW", close_calendar)

    # Add the Calendar widget
    cal = Calendar(calendar_window, date_pattern="yyyy-mm-dd", background="lightblue", foreground="black")
    cal.pack(pady=10)

    # Add a "Select Date" button to confirm selection
    tk.Button(
        calendar_window, text="Select Date", command=select_date, bg="#448aff", fg="white", font=("Arial", 12)
    ).pack(pady=5)



def init_db():
    """
    Initializes the database and creates the transactions table if it doesn't already exist.
    """
    with sqlite3.connect("financial_tracker.db") as db_connection:
        cursor = db_connection.cursor()
        cursor.execute('''
        CREATE TABLE IF NOT EXISTS transactions (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            date TEXT NOT NULL,
            category TEXT NOT NULL,
            subtype TEXT NOT NULL,
            description TEXT NOT NULL,
            amount REAL NOT NULL
        )
        ''')
        db_connection.commit()


def update_subtypes(event):
    """
    Updates the available subtypes based on the selected category.
    """
    selected_category = category_var.get()
    subtypes = category_to_subtypes.get(selected_category, [])
    subtype_var.set("")  # Clear current selection
    subtype_menu.config(values=subtypes)


def add_transaction():
    """
    Adds a new transaction to the database.
    """
    date = date_entry.get()
    category = category_var.get()
    subtype = subtype_var.get()
    description = description_entry.get()
    amount = amount_entry.get()

    if not date or not category or not subtype or not description or not amount:
        messagebox.showerror("Input Error", "All fields must be filled!")
        return

    try:
        amount = float(amount)
    except ValueError:
        messagebox.showerror("Input Error", "Amount must be a valid number!")
        return

    with sqlite3.connect("financial_tracker.db") as db_connection:
        cursor = db_connection.cursor()
        cursor.execute('''
        INSERT INTO transactions (date, category, subtype, description, amount)
        VALUES (?, ?, ?, ?, ?)
        ''', (date, category, subtype, description, amount))
        db_connection.commit()

    reset_inputs()
    load_transactions_async()
    calculate_balance_async()
    messagebox.showinfo("Success", "Transaction added successfully!")


def load_transactions(limit=100, offset=0):
    """
    Fetches and displays transactions from the database.
    """
    for row in transaction_table.get_children():
        transaction_table.delete(row)

    with sqlite3.connect("financial_tracker.db") as db_connection:
        cursor = db_connection.cursor()
        cursor.execute(
            f"SELECT id, date, category, subtype, description, amount FROM transactions LIMIT {limit} OFFSET {offset}")
        rows = cursor.fetchall()

    for row in rows:
        transaction_table.insert("", "end", values=row)


def load_transactions_async(limit=100, offset=0):
    """
    Loads transactions in a background thread to avoid blocking the GUI.
    """
    Thread(target=load_transactions, args=(limit, offset)).start()


def calculate_balance():
    """
    Computes the total balance by summing all transaction amounts.
    """
    with sqlite3.connect("financial_tracker.db") as db_connection:
        cursor = db_connection.cursor()
        cursor.execute("SELECT SUM(amount) FROM transactions")
        result = cursor.fetchone()

    balance = result[0] if result[0] is not None else 0.0
    balance_label.config(text=f"Current Balance: ${balance:.2f}")


def calculate_balance_async():
    """
    Calculates the balance in a background thread to improve GUI responsiveness.
    """
    Thread(target=calculate_balance).start()


def delete_transaction():
    """
    Deletes selected transactions from the database.
    """
    selected_items = transaction_table.selection()
    if not selected_items:
        messagebox.showwarning("Delete Transaction", "No transaction selected!")
        return

    with sqlite3.connect("financial_tracker.db") as db_connection:
        cursor = db_connection.cursor()
        for item in selected_items:
            transaction_id = transaction_table.item(item, "values")[0]
            cursor.execute("DELETE FROM transactions WHERE id=?", (transaction_id,))
            transaction_table.delete(item)

    calculate_balance_async()
    messagebox.showinfo("Success", "Selected transaction(s) deleted.")


def reset_inputs():
    """
    Clears the input fields.
    """
    date_entry.delete(0, tk.END)
    category_var.set("")
    subtype_var.set("")
    description_entry.delete(0, tk.END)
    amount_entry.delete(0, tk.END)


app = tk.Tk()
app.title("Abdullah's Financial Tracker")
app.geometry("850x600")
app.resizable(True, True)

# Configure grid resizing
app.grid_rowconfigure(0, weight=1)
app.grid_columnconfigure(0, weight=1)

# Main Frame
main_frame = tk.Frame(app, bg="white")
main_frame.grid(row=0, column=0, sticky="nsew")

# Configure resizing within the main frame
main_frame.grid_rowconfigure(6, weight=1)  # Allow table area to expand
main_frame.grid_columnconfigure(1, weight=1)

# Category-to-Subtype Mapping
category_to_subtypes = {
    "Food": ["Groceries", "Restaurant", "Snacks"],
    "Medicine": ["Pharmacy", "Vitamins", "Doctor Fees"],
    "Loans": ["Personal Loan", "Mortgage", "Business Loan"],
    "Sales": ["Retail", "Online", "Wholesale"],
    "Rents": ["House Rent", "Car Rent", "Equipment Rent"]
}

# Input Fields
tk.Label(main_frame, text="Date:", font=("Arial", 12), bg="white", fg="black").grid(row=0, column=0, padx=10, pady=5)
date_entry = tk.Entry(main_frame, font=("Arial", 12), bg="white")
date_entry.grid(row=0, column=1, padx=10, pady=5, sticky="ew")
tk.Button(
    main_frame, text="📅", command=lambda: open_calendar(date_entry), font=("Arial", 12), bg="#448aff", fg="white"
).grid(row=0, column=2, padx=5)

tk.Label(main_frame, text="Category:", font=("Arial", 12), bg="white", fg="black").grid(row=1, column=0, padx=10,
                                                                                        pady=5)
category_var = tk.StringVar()
category_menu = ttk.Combobox(main_frame, textvariable=category_var, font=("Arial", 12),
                             values=list(category_to_subtypes.keys()))
category_menu.grid(row=1, column=1, padx=10, pady=5, sticky="ew")
category_menu.bind("<<ComboboxSelected>>", update_subtypes)

tk.Label(main_frame, text="Subtype:", font=("Arial", 12), bg="white", fg="black").grid(row=2, column=0, padx=10, pady=5)
subtype_var = tk.StringVar()
subtype_menu = ttk.Combobox(main_frame, textvariable=subtype_var, font=("Arial", 12))
subtype_menu.grid(row=2, column=1, padx=10, pady=5, sticky="ew")

tk.Label(main_frame, text="Description:", font=("Arial", 12), bg="white", fg="black").grid(row=3, column=0, padx=10,
                                                                                           pady=5)
description_entry = tk.Entry(main_frame, font=("Arial", 12), bg="white", width=30)
description_entry.grid(row=3, column=1, columnspan=2, padx=10, pady=5, sticky="ew")

tk.Label(main_frame, text="Amount ($):", font=("Arial", 12), bg="white", fg="black").grid(row=4, column=0, padx=10,
                                                                                          pady=5)
amount_entry = tk.Spinbox(main_frame, from_=0.0, to=100000.0, increment=1.0, font=("Arial", 12), bg="white")
amount_entry.grid(row=4, column=1, padx=10, pady=5, sticky="ew")

tk.Button(
    main_frame, text="Add Transaction", command=add_transaction, font=("Arial", 12), bg="#4CAF50", fg="white"
).grid(row=5, column=0, pady=10)
tk.Button(
    main_frame, text="Delete Transaction", command=delete_transaction, font=("Arial", 12), bg="#F44336", fg="white"
).grid(row=5, column=1, pady=10)

# Transaction Table
transaction_table = ttk.Treeview(main_frame, columns=("ID", "Date", "Category", "Subtype", "Description", "Amount"),
                                 show="headings")
transaction_table.heading("ID", text="ID")
transaction_table.heading("Date", text="Date")
transaction_table.heading("Category", text="Category")
transaction_table.heading("Subtype", text="Subtype")
transaction_table.heading("Description", text="Description")
transaction_table.heading("Amount", text="Amount ($)")
transaction_table.grid(row=6, column=0, columnspan=3, sticky="nsew", pady=10, padx=10)

# Scrollbar for Table
scrollbar = ttk.Scrollbar(main_frame, orient="vertical", command=transaction_table.yview)
scrollbar.grid(row=6, column=3, sticky="ns")
transaction_table.configure(yscrollcommand=scrollbar.set)

# Balance Display
balance_label = tk.Label(main_frame, text="Current Balance: $0.00", font=("Arial", 16), bg="white", fg="black")
balance_label.grid(row=7, column=0, columnspan=2, sticky="ew")

# Database and startup
init_db()
load_transactions_async()
calculate_balance_async()

app.mainloop()
