# Expense-test
import streamlit as st
import sqlite3
import pandas as pd
from datetime import date

# 1. Local Database Setup
def get_db():
    conn = sqlite3.connect('roommates.db', check_same_thread=False)
    conn.execute('''
        CREATE TABLE IF NOT EXISTS expenses (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            payer TEXT,
            amount REAL,
            date TEXT,
            purpose TEXT,
            category TEXT,
            approved_by TEXT
        )
    ''')
    conn.commit()
    return conn

conn = get_db()

# 2. Roommates & User Switcher
st.title("🏠 Expense Tracker (Test Mode)")
roommates = ["Alice", "Bob", "Charlie"]

# Switch between friends here to test approvals from different perspectives
current_user = st.selectbox("👤 Acting as:", roommates)
st.divider()

categories = ["Rent", "Food & Groceries", "Utilities", "Household Supplies", "Other"]

# 3. Add Expense Form
with st.expander("➕ Add New Expense", expanded=False):
    with st.form("expense_form"):
        col1, col2 = st.columns(2)
        with col1:
            payer = st.selectbox("Who paid?", roommates, index=roommates.index(current_user))
            amount = st.number_input("Amount", min_value=0.01, step=1.0)
            date_paid = st.date_input("Date", date.today())
        with col2:
            category = st.selectbox("Category", categories)
            purpose = st.text_input("Purpose (e.g., Grocery run)")
        
        submitted = st.form_submit_button("Log Expense")
        if submitted:
            conn.execute(
                "INSERT INTO expenses (payer, amount, date, purpose, category, approved_by) VALUES (?, ?, ?, ?, ?, ?)",
                (payer, amount, str(date_paid), purpose, category, payer)
            )
            conn.commit()
            st.success("Expense logged! Automatically approved by the payer.")
            st.rerun()

# 4. Read Data
df = pd.read_sql_query("SELECT * FROM expenses ORDER BY date DESC", conn)

if not df.empty:
    df['approved_by'] = df['approved_by'].fillna("")
    
    def is_fully_approved(approvers_str):
        approvers = [a.strip() for a in approvers_str.split(',') if a.strip()]
        return len(set(approvers)) == len(roommates)
        
    df['is_approved'] = df['approved_by'].apply(is_fully_approved)
    
    df_pending = df[~df['is_approved']]
    df_approved = df[df['is_approved']]

    # Pending Approvals Queue
    st.subheader("⏳ Pending Approvals")
    if not df_pending.empty:
        for _, row in df_pending.iterrows():
            approvers = [a.strip() for a in row['approved_by'].split(',') if a.strip()]
            missing = [r for r in roommates if r not in approvers]
            
            with st.container():
                col1, col2 = st.columns([3, 1])
                col1.write(f"**{row['payer']}** logged **${row['amount']}** for *{row['purpose']}*")
                col1.caption(f"Waiting on: {', '.join(missing)}")
                
                if current_user not in approvers:
                    if col2.button("Approve", key=f"app_btn_{row['id']}", type="primary"):
                        new_approvers = row['approved_by'] + f",{current_user}"
                        conn.execute("UPDATE expenses SET approved_by = ? WHERE id = ?", (new_approvers, row['id']))
                        conn.commit()
                        st.rerun()
                else:
                    col2.button("Approved ✓", key=f"done_{row['id']}", disabled=True)
                st.divider()
    else:
        st.info("No pending approvals.")

    # Approved Ledger
    st.subheader("📊 Settled Ledger")
    if not df_approved.empty:
        st.dataframe(df_approved.drop(columns=['id', 'approved_by', 'is_approved']), use_container_width=True)
        
        st.subheader("💰 Balance Settlement")
        summary = df_approved.groupby('payer')['amount'].sum().reset_index()
        total_spent = df_approved['amount'].sum()
        per_person_share = total_spent / len(roommates)
        
        st.write(f"**Total Settled Spend:** ${total_spent:.2f}")
        st.write(f"**Target Share Each:** ${per_person_share:.2f}")
        
        for roomie in roommates:
            paid = summary[summary['payer'] == roomie]['amount'].sum() if roomie in summary['payer'].values else 0.0
            balance = paid - per_person_share
            if balance > 0:
                st.success(f"**{roomie}** is owed ${balance:.2f}")
            elif balance < 0:
                st.error(f"**{roomie}** owes ${abs(balance):.2f}")
            else:
                st.info(f"**{roomie}** is settled up!")
    else:
        st.write("Expenses will appear here once approved by all 3 friends.")
else:
    st.info("No entries yet. Open 'Add New Expense' above to log your first test transaction.")
