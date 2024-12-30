from flask import Flask, jsonify, request
import hashlib
import json
import time
import random

class Block:
    def __init__(self, index, previous_hash, timestamp, data, hash, nonce):
        self.index = index
        self.previous_hash = previous_hash
        self.timestamp = timestamp
        self.data = data
        self.hash = hash
        self.nonce = nonce

    def __str__(self):
        return f'Block {self.index} [previous_hash: {self.previous_hash}, timestamp: {self.timestamp}, data: {self.data}, hash: {self.hash}, nonce: {self.nonce}]'

class Blockchain:
    def __init__(self):
        self.chain = []
        self.current_transactions = []
        self.create_genesis_block()

    def create_genesis_block(self):
        genesis_block = Block(0, "0", int(time.time()), "Genesis Block", self.calculate_hash(0, "0", int(time.time()), "Genesis Block", 0), 0)
        self.chain.append(genesis_block)

    def calculate_hash(self, index, previous_hash, timestamp, data, nonce):
        value = str(index) + previous_hash + str(timestamp) + str(data) + str(nonce)
        return hashlib.sha256(value.encode()).hexdigest()

    def add_transaction(self, sender, recipient, amount):
        self.current_transactions.append({
            'sender': sender,
            'recipient': recipient,
            'amount': amount,
        })

    def mine_block(self):
        previous_block = self.chain[-1]
        index = previous_block.index + 1
        timestamp = int(time.time())
        nonce = self.proof_of_work(previous_block.nonce)
        hash = self.calculate_hash(index, previous_block.hash, timestamp, self.current_transactions, nonce)
        new_block = Block(index, previous_block.hash, timestamp, self.current_transactions, hash, nonce)
        self.chain.append(new_block)
        self.current_transactions = []
        return new_block

    def proof_of_work(self, last_nonce):
        nonce = 0
        while not self.valid_proof(last_nonce, nonce):
            nonce += 1
        return nonce

    def valid_proof(self, last_nonce, nonce):
        guess = f'{last_nonce}{nonce}'.encode()
        guess_hash = hashlib.sha256(guess).hexdigest()
        return guess_hash[:4] == "0000"  # Сложность: 4 нуля на старте хеша

app = Flask(__name__)
blockchain = Blockchain()

@app.route('/transactions/new', methods=['POST'])
def new_transaction():
    values = request.json
    required = ['sender', 'recipient', 'amount']
    if not all(k in values for k in required):
        return 'Missing values', 400

    blockchain.add_transaction(values['sender'], values['recipient'], values['amount'])
    return 'Transaction will be added to the next block', 201

@app.route('/mine', methods=['GET'])
def mine():
    block = blockchain.mine_block()
    response = {
        'message': 'New Block Mined',
        'index': block.index,
        'transactions': block.data,
        'nonce': block.nonce,
        'previous_hash': block.previous_hash,
        'timestamp': block.timestamp,
    }
    return jsonify(response), 200

@app.route('/chain', methods=['GET'])
def get_chain():
    response = {
        'chain': [str(block) for block in blockchain.chain],
        'length': len(blockchain.chain),
    }
    return jsonify(response), 200

if __name__ == '__main__':
    app.run(debug=True)
