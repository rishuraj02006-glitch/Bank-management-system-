import sqlite3
import datetime
from getpass import getpass
import random

DB_NAME = "bank.db"

class BankSystem:
    def __init__(self):
        self.conn = sqlite3.connect(DB_NAME)
        self.create_tables()

    def create_tables(self):
        cur = self.conn.cursor()
        cur.execute("""
            CREATE TABLE IF NOT EXISTS accounts (
                acc_no INTEGER PRIMARY KEY,
                name TEXT NOT NULL,
                pin TEXT NOT NULL,
                balance REAL DEFAULT 0,
                created_at TEXT DEFAULT CURRENT_TIMESTAMP
            )
        """)
        cur.execute("""
            CREATE TABLE IF NOT EXISTS transactions (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                acc_no INTEGER,
                type TEXT,
                amount REAL,
                balance_after REAL,
                timestamp TEXT DEFAULT CURRENT_TIMESTAMP,
                note TEXT,
                FOREIGN KEY(acc_no) REFERENCES accounts(acc_no)
            )
        """)
        self.conn.commit()

    def generate_acc_no(self):
        while True:
            acc_no = random.randint(100000, 999999)
            cur = self.conn.cursor()
            cur.execute("SELECT 1 FROM accounts WHERE acc_no=?", (acc_no,))
            if not cur.fetchone():
                return acc_no

    def create_account(self, name, pin):
        if len(pin)!= 4 or not pin.isdigit():
            return "PIN must be 4 digits"

        acc_no = self.generate_acc_no()
        cur = self.conn.cursor()
        cur.execute("INSERT INTO accounts (acc_no, name, pin, balance) VALUES (?,?,?, 0)",
                    (acc_no, name, pin))
        self.conn.commit()
        return f"Account created. Your account number: {acc_no}"

    def authenticate(self, acc_no, pin):
        cur = self.conn.cursor()
        cur.execute("SELECT name, balance FROM accounts WHERE acc_no=? AND pin=?", (acc_no, pin))
        return cur.fetchone() # Returns (name, balance) or None

    def get_balance(self, acc_no):
        cur = self.conn.cursor()
        cur.execute("SELECT balance FROM accounts WHERE acc_no=?", (acc_no,))
        res = cur.fetchone()
        return res[0] if res else None

    def log_transaction(self, acc_no, trans_type, amount, note=""):
        balance = self.get_balance(acc_no)
        cur = self.conn.cursor()
        cur.execute("""
            INSERT INTO transactions (acc_no, type, amount, balance_after, note)
            VALUES (?,?,?,?,?)
        """, (acc_no, trans_type, amount, balance, note))
        self.conn.commit()

    def deposit(self, acc_no, amount):
        if amount <= 0: return "Amount must be positive"
        cur = self.conn.cursor()
        cur.execute("UPDATE accounts SET balance = balance +? WHERE acc_no=?", (amount, acc_no))
        self.conn.commit()
        self.log_transaction(acc_no, "DEPOSIT", amount)
        return f"Deposited ${amount:.2f}. New balance: ${self.get_balance(acc_no):.2f}"

    def withdraw(self, acc_no, amount):
        if amount <= 0: return "Amount must be positive"
        if self.get_balance(acc_no) < amount: return "Insufficient funds"

        cur = self.conn.cursor()
        cur.execute("UPDATE accounts SET balance = balance -? WHERE acc_no=?", (amount, acc_no))
        self.conn.commit()
        self.log_transaction(acc_no, "WITHDRAW", amount)
        return f"Withdrew ${amount:.2f}. New balance: ${self.get_balance(acc_no):.2f}"

    def transfer(self, from_acc, to_acc, amount):
        if amount <= 0: return "Amount must be positive"
        if self.get_balance(from_acc) < amount: return "Insufficient funds"
        if not self.account_exists(to_acc): return "Recipient account not found"
        if from_acc == to_acc: return "Cannot transfer to same account"

        cur = self.conn.cursor()
        cur.execute("UPDATE accounts SET balance = balance -? WHERE acc_no=?", (amount, from_acc))
        cur.execute("UPDATE accounts SET balance = balance +? WHERE acc_no=?", (amount, to_acc))
        self.conn.commit()

        self.log_transaction(from_acc, "TRANSFER_OUT", amount, f"To {to_acc}")
        self.log_transaction(to_acc, "TRANSFER_IN", amount, f"From {from_acc}")
        return f"Transferred ${amount:.2f} to {to_acc}. New balance: ${self.get_balance(from_acc):.2f}"

    def account_exists(self, acc_no):
        cur = self.conn.cursor()
        cur.execute("SELECT 1 FROM accounts WHERE acc_no=?", (acc_no,))
        return cur.fetchone() is not None

    def transaction_history(self, acc_no, limit=10):
        cur = self.conn.cursor()
        cur.execute("""
            SELECT type, amount, balance_after, timestamp, note
            FROM transactions WHERE acc_no=?
            ORDER BY timestamp DESC LIMIT?
        """, (acc_no, limit))
        return cur.fetchall()



def main():
    bank = BankSystem()

    while True:
        print("\n=== Bank Management System ===")
        print("1. Create Account")
        print("2. Login")
        print("3. Exit")
        choice = input("Select: ")

        if choice == "1":
            name = input("Enter your name: ")
            pin = getpass("Set 4-digit PIN: ")
            print(bank.create_account(name, pin))

        elif choice == "2":
            try:
                acc_no = int(input("Account number: "))
                pin = getpass("PIN: ")
            except ValueError:
                print("Invalid input")
                continue

            user = bank.authenticate(acc_no, pin)
            if not user:
                print("Authentication failed")
                continue

            name, _ = user
            print(f"\nWelcome, {name}")

            while True:
                print("\n1. Check Balance 2. Deposit 3. Withdraw")
                print("4. Transfer 5. Transaction History 6. Logout")
                op = input("Select: ")

                if op == "1":
                    print(f"Balance: ${bank.get_balance(acc_no):.2f}")
                elif op == "2":
                    amt = float(input("Amount to deposit: $"))
                    print(bank.deposit(acc_no, amt))
                elif op == "3":
                    amt = float(input("Amount to withdraw: $"))
                    print(bank.withdraw(acc_no, amt))
                elif op == "4":
                    to_acc = int(input("Recipient account no: "))
                    amt = float(input("Amount: $"))
                    print(bank.transfer(acc_no, to_acc, amt))
                elif op == "5":
                    history = bank.transaction_history(acc_no)
                    print("\n**Transaction History**")
                    print("| Type | Amount | Balance | Date | Note |")
                    print("| --- | --- | --- |")
                    for t in history:
                        print(f"| {t[0]} | ${t[1]:.2f} | ${t[2]:.2f} | {t[3][:16]} | {t[4]} |")
                elif op == "6":
                    break
                else:
                    print("Invalid option")

        elif choice == "3":
            break
        else:
            print("Invalid choice")

if __name__ == "__main__":
    main()
 Bank-management-system-