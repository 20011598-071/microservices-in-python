# microservices-in-python
microservices-in-python

import json
import sqlite3

from flask import Flask, request, jsonify
from flask_sqlalchemy import SQLAlchemy

app = Flask(__name__)
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///transactions.db'
db = SQLAlchemy(app)

# ... Rest of the code remains the same ...
# import json

from flask import Flask, request, jsonify
from flask_sqlalchemy import SQLAlchemy

app = Flask(__name__)
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///transactions.db'
db = SQLAlchemy(app)
conn = sqlite3.connect('transactions.db')
class Transaction(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    account_number = db.Column(db.String(20), nullable=False)
    transaction_type = db.Column(db.String(10), nullable=False)
    amount = db.Column(db.Float, nullable=False)

def create_database():
    with app.app_context():
        db.create_all()

def load_transaction_data(file_path):
    with open(file_path, 'r') as json_file:
        data = json.load(json_file)
    return data

def store_transactions(transactions_data):
    with app.app_context():
        for transaction_data in transactions_data:
            account_number = transaction_data.get('account_number')
            transaction_type = transaction_data.get('transaction_type')
            amount = transaction_data.get('amount')

            if account_number and transaction_type and amount:
                new_transaction = Transaction(account_number=account_number,
                                              transaction_type=transaction_type,
                                              amount=amount)
                db.session.add(new_transaction)
        db.session.commit()

@app.route('/transactions', methods=['GET', 'POST'])
def transactions():
    if request.method == 'GET':
        transactions_list = Transaction.query.all()
        transactions_data = []
        for transaction in transactions_list:
            transaction_data = {
                'id': transaction.id,
                'account_number': transaction.account_number,
                'transaction_type': transaction.transaction_type,
                'amount': transaction.amount
            }
            transactions_data.append(transaction_data)
        return jsonify(transactions_data), 200
    elif request.method == 'POST':
        data = request.get_json()
        account_number = data.get('account_number')
        transaction_type = data.get('transaction_type')
        amount = data.get('amount')

        if not all([account_number, transaction_type, amount]):
            return jsonify({'message': 'Invalid data provided'}), 400

        new_transaction = Transaction(account_number=account_number,
                                      transaction_type=transaction_type,
                                      amount=amount)
        db.session.add(new_transaction)
        db.session.commit()

        return jsonify({'message': 'Transaction created successfully'}), 201

@app.route('/update_balance', methods=['POST'])
def update_balance():
    data = request.get_json()
    account_number = data.get('account_number')
    transaction_type = data.get('transaction_type')
    amount = data.get('amount')

    if not all([account_number, transaction_type, amount]):
        return jsonify({'message': 'Invalid data provided'}), 400

    if transaction_type not in ('credit', 'debit'):
        return jsonify({'message': 'Invalid transaction type'}), 400

    # Fetch the account from the database based on the account_number
    account = Transaction.query.filter_by(account_number=account_number).first()
    if not account:
        return jsonify({'message': 'Account not found'}), 404

    if transaction_type == 'credit':
        account.amount += amount
    else:
        if account.amount < amount:
            return jsonify({'message': 'Insufficient balance'}), 400
        account.amount -= amount

    db.session.commit()
    return jsonify({'message': 'Balance updated successfully'}), 200

@app.route('/', methods=['GET'])
def home():
    return jsonify({'message': 'Welcome to the Bank Transaction Microservice!'}), 200

if __name__ == '__main__':
    transaction_data_file = 'transaction_data.json'
    transactions_data = load_transaction_data(transaction_data_file)

    create_database()
    store_transactions(transactions_data)

    app.run(debug=True, host='0.0.0.0')
