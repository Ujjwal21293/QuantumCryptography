# QuantumCryptography

# Quantum Computing
In this notebook we are going to introduce the theory of Quantum Cryptography and one of its possible applications.
This notebook consists in a possible implementation of the BB84 cryptographic protocol on a quantum computer, reproducing Quantum Key Distribution and eavesdropper detection. It makes use of IBM's Qiskit, a python library that can manipulate quantum circuits, either via a simulation or a real execution on IBM's backend.
# Quantum Key Distribution
In 1984, building on the work of Charles Bennett and Gilles Brassard developed the first quantum cryptographic protocol, which goes under the codename of BB84.

Suppose that Alice and Bob are connected via a quantum channel that they can use to exchange qubits. This channel is not used directly to send a private message, but only to exchange random qubits that after processing will compose the encryption key.

Alice produces an initial key, selecting a sequence of random bits, '0' and '1', and picking a sequence of polarization eigenstates, with respect to a randomly chosen basis between: rectilinear and diagonal. Now, using the quantum channel, she sends the stream of qubits to Bob, who is unaware of the basis used by Alice for the encoding. Bob receives these qubits prepared in a certain polarization eigenstate but, due to the no-cloning theorem, he is unable to recognize which basis Alice used, because he cannot distinguish non-orthogonal states with a single measurement. Nonetheless he proceeds anyway with measuring each photon's polarization using a basis chosen randomly (between rectilinear and diagonal), and he keeps a note of the measurement result and the associated basis that he used in a report table. Bob will pick the right basis, the same that Alice picked, about 
1/2 of the times, and the wrong basis about 1/2 of the times.

For this reason Alice and Bob decide to discard the wrong key obtained via measurements made in the wrong basis, since they are not reliable. The price for this action is that the key will lose about 1/2 of its length, but the payoff is that they don't need to unveil their measurements, they just need to compare their tables, where they recorded the basis chosen, and they do that after the measurement has occurred. So they open the classical channel and only now Alice tells Bob which basis she used to encode the key; they compare the tables. They'll obtain a perfectly correlated key.

Now, what will happen when there is eavesdropping in communication? Suppose, Eve wants to know the information of Alice, what she can do is that she can use any one of her basis to measure the qubits. In this process either she'll get the correct order of the qubit or her measurement will alter the information of the qubit.


## Importing Essential Module

```python
# Import numpy for random number generation
import numpy as np

# importing Qiskit
from qiskit import QuantumCircuit, ClassicalRegister, QuantumRegister, execute, BasicAer

# Import basic plotting tools
from qiskit.tools.visualization import plot_histogram
```
Now we'll create the number of qubit to 10.
```python
n = 16  
qr = QuantumRegister(n, name='qr')
cr = ClassicalRegister(n, name='cr')
```
We create Alice's Quantum Circuit, made of n qubits.We use `randint` from numpy to generate a random number in the available range which will be our key and then we write the resulted number in binary and we memorize the key in a proper variable.
```python
alice = QuantumCircuit(qr, cr, name='Alice')
alice_key = np.random.randint(0, high=2**n)
alice_key = np.binary_repr(alice_key, n)
```
we encode it in Alice's circuit, initializing her qubits to the computational basis: {|0>,|1>}, according to the value bit. Then we apply a rotation to about half of these qubits, so that about 1/2 of them will now be in one of the eigenstates of the diagonal basis.
```python
for index, digit in enumerate(alice_key):
    if digit == '1':
        alice.x(qr[index]) 
        
# Switch randomly about half qubits to diagonal basis
alice_table = []        
for index in range(len(qr)):       
    if 0.5 < np.random.random():   
        alice.h(qr[index])         # change to diagonal basis
        alice_table.append('X')    # character for diagonal basis
    else:
        alice_table.append('Z')
```
we create quantum circuit, which we call bob and initialize Bob's initial state to Alice's output state. 
```python
def SendState(qc1, qc2, qc1_name):
    ''' This function takes the output of a circuit qc1 (made up only of x and 
        h gates and initializes another circuit qc2 with the same state
    ''' 
    # Quantum state is retrieved 
    qs = qc1.qasm().split(sep=';')[4:-1]

    for index, instruction in enumerate(qs):
        qs[index] = instruction.lstrip()

    for instruction in qs:
        if instruction[0] == 'x':
            old_qr = int(instruction[5:-1])
            qc2.x(qr[old_qr])
        elif instruction[0] == 'h':
            old_qr = int(instruction[5:-1])
            qc2.h(qr[old_qr])
        elif instruction[0] == 'm': # exclude measuring:
            pass
        else:
            raise Exception('Unable to parse')
```
Now we create Bob's circuit and send Alice's qubits to bob.
```python
bob = QuantumCircuit(qr, cr, name='Bob')

SendState(alice, bob, 'Alice')    

# Bob doesn't know which basis to use
bob_table = []
for index in range(len(qr)): 
    if 0.5 < np.random.random():  # 50% chance...
        bob.h(qr[index])       
        bob_table.append('X')
    else:
        bob_table.append('Z')
```
Bob can now go ahead and measure all his qubits and store the measurement in the classical register.
```python
for index in range(len(qr)): 
    bob.measure(qr[index], cr[index])
backend = BasicAer.get_backend('qasm_simulator')    
execute(bob, backend=backend, shots=1).result()
```
Now we could know in which qubits both Bob and Alice has done same basis measurement or not.
```python
keep = []
discard = []
for qubit, basis in enumerate(zip(alice_table, bob_table)):
    if basis[0] == basis[1]:
        print("Same choice for qubit: {}, basis: {}" .format(qubit, basis[0])) 
        keep.append(qubit)
    else:
        print("Different choice for qubit: {}, Alice has {}, Bob has {}" .format(qubit, basis[0], basis[1]))
        discard.append(qubit)
```
After that they'll talk over the classical channel and separate out in which qubits they have done the same measurement and in which they haven't they'll discard them.

Now what will happen when there's Eve and she'll measure the qubits randomly.
```python
eve = QuantumCircuit(qr, cr, name='Eve')
SendState(alice, eve, 'Alice') 
eve_table = []
for index in range(len(qr)): 
    if 0.5 < np.random.random(): 
        eve.h(qr[index])        
        eve_table.append('X')
    else:
        eve_table.append('Z')
```

```python
for index in range(len(qr)): 
    eve.measure(qr[index], cr[index])
backend = BasicAer.get_backend('qasm_simulator')    
result = execute(eve, backend=backend, shots=1).result()

eve_key = list(result.get_counts(eve))[0]
eve_key = eve_key[::-1]
```
Now we'll know in which eve has succesfully get the measurement and in whcih she didn't.
```python
for qubit, basis in enumerate(zip(alice_table, eve_table)):
    if basis[0] == basis[1]:
        print("Same choice for qubit: {}, basis: {}" .format(qubit, basis[0]))
    else:
        print("Different choice for qubit: {}, Alice has {}, Eve has {}" .format(qubit, basis[0], basis[1]))
        if eve_key[qubit] == alice_key[qubit]:
            eve.h(qr[qubit])
        else:
            if basis[0] == 'X' and basis[1] == 'Z':
                eve.h(qr[qubit])
                eve.x(qr[qubit])
            else:
                eve.x(qr[qubit])
                eve.h(qr[qubit])
```
Now she'll alter the basis and send it to Bob
```python
SendState(eve, bob, 'Eve')
          
bob_table = []
for index in range(len(qr)): 
    if 0.5 < np.random.random(): 
        bob.h(qr[index])      
        bob_table.append('X')
    else:
        bob_table.append('Z')
          
for index in range(len(qr)): 
    bob.measure(qr[index], cr[index])
          
execute(bob, backend=backend, shots=1).result()
```
we'll know the presence of eve.
```python
keep = []
discard = []
for qubit, basis in enumerate(zip(alice_table, bob_table)):
    if basis[0] == basis[1]:
        print("Same choice for qubit: {}, basis: {}" .format(qubit, basis[0])) 
        keep.append(qubit)
    else:
        print("Different choice for qubit: {}, Alice has {}, Bob has {}" .format(qubit, basis[0], basis[1]))
        discard.append(qubit)
        
acc = 0
for bit in zip(alice_key, bob_key):
    if bit[0] == bit[1]:
        acc += 1

print('\nPercentage of qubits to be discarded according to table comparison: ', len(keep)/n)
print('Measurement convergence by additional chance: ', acc/n)  

new_alice_key = [alice_key[qubit] for qubit in keep]
new_bob_key = [bob_key[qubit] for qubit in keep]

acc = 0
for bit in zip(new_alice_key, new_bob_key):
    if bit[0] == bit[1]:
        acc += 1        
        
print('\nPercentage of similarity between the keys: ', acc/len(new_alice_key)) 

if (acc//len(new_alice_key) == 1):
    print("\nKey exchange has been successfull")
    print("New Alice's key: ", new_alice_key)
    print("New Bob's key: ", new_bob_key)
else:
    print("\nKey exchange has been tampered! Check for eavesdropper or try again")
    print("New Alice's key is invalid: ", new_alice_key)
    print("New Bob's key is invalid: ", new_bob_key)
```
## Acknowledgements

 - [github document](https://github.com/qiskit-community/qiskit-community-tutorials/blob/master/awards/teach_me_qiskit_2018/cryptography/Cryptography.ipynb)

