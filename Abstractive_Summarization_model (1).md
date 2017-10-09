

```python
import numpy as np
from __future__ import division

filename = 'glove.6B.50d.txt'
def loadGloVe(filename):
    vocab = []
    embd = []
    file = open(filename,'r')
    for line in file.readlines():
        row = line.strip().split(' ')
        vocab.append(row[0])
        embd.append(row[1:])
    print('Loaded GloVe!')
    file.close()
    return vocab,embd
vocab,embd = loadGloVe(filename)

embedding = np.asarray(embd)
embedding = embedding.astype(np.float32)

word_vec_dim = len(embedding[0])
#Pre-trained GloVe embedding
```

    Loaded GloVe!



```python
def np_nearest_neighbour(x):
    #returns array in embedding that's most similar (in terms of cosine similarity) to x
        
    xdoty = np.multiply(embedding,x)
    xdoty = np.sum(xdoty,1)
    xlen = np.square(x)
    xlen = np.sum(xlen,0)
    xlen = np.sqrt(xlen)
    ylen = np.square(embedding)
    ylen = np.sum(ylen,1)
    ylen = np.sqrt(ylen)
    xlenylen = np.multiply(xlen,ylen)
    cosine_similarities = np.divide(xdoty,xlenylen)

    return embedding[np.argmax(cosine_similarities)]


def word2vec(word):  # converts a given word into its vector representation
    if word in vocab:
        return embedding[vocab.index(word)]
    else:
        return embedding[vocab.index('unk')]

def vec2word(vec):   # converts a given vector representation into the represented word 
    for x in xrange(0, len(embedding)):
        if np.array_equal(embedding[x],np.asarray(vec)):
            return vocab[x]
    return vec2word(np_nearest_neighbour(np.asarray(vec)))
```


```python
import pickle

with open ('vec_summaries', 'rb') as fp:
    vec_summaries = pickle.load(fp)

with open ('vec_texts', 'rb') as fp:
    vec_texts = pickle.load(fp)
    
```


```python
with open ('vocab_limit', 'rb') as fp:
    vocab_limit = pickle.load(fp)

with open ('embd_limit', 'rb') as fp:
    embd_limit = pickle.load(fp)
    
```


```python
#DIAGNOSIS

count = 0

LEN = 7

for summary in vec_summaries:
    if len(summary)-1>LEN:
        count = count + 1
print "Percentage of dataset with summary length beyond "+str(LEN)+": "+str((count/len(vec_summaries))*100)+"% "

count = 0

D = 10 

window_size = 2*D+1

for text in vec_texts:
    if len(text)<window_size+1:
        count = count + 1
print "Percentage of dataset with text length less that window size: "+str((count/len(vec_texts))*100)+"% "

count = 0

LEN = 80

for text in vec_texts:
    if len(text)>LEN:
        count = count + 1
print "Percentage of dataset with text length more than "+str(LEN)+": "+str((count/len(vec_texts))*100)+"% "
```

    Percentage of dataset with summary length beyond 7: 16.146% 
    Percentage of dataset with text length less that window size: 2.258% 
    Percentage of dataset with text length more than 80: 40.412% 



```python
MAX_SUMMARY_LEN = 7
MAX_TEXT_LEN = 80

#D is a major hyperparameters. Windows size for local attention will be 2*D+1
D = 10

window_size = 2*D+1

#REMOVE DATA WHOSE SUMMARIES ARE TOO BIG
#OR WHOSE TEXT LENGTH IS TOO BIG
#OR WHOSE TEXT LENGTH IS SMALLED THAN WINDOW SIZE

vec_summaries_reduced = []
vec_texts_reduced = []

i = 0
for summary in vec_summaries:
    if len(summary)-1<=MAX_SUMMARY_LEN and len(vec_texts[i])>=window_size and len(vec_texts[i])<=MAX_TEXT_LEN:
        vec_summaries_reduced.append(summary)
        vec_texts_reduced.append(vec_texts[i])
    i=i+1
```


```python
train_len = int((.7)*len(vec_summaries_reduced))

train_texts = vec_texts_reduced[0:train_len]
train_summaries = vec_summaries_reduced[0:train_len]

val_len = int((.15)*len(vec_summaries_reduced))

val_texts = vec_texts_reduced[train_len:train_len+val_len]
val_summaries = vec_summaries_reduced[train_len:train_len+val_len]

test_texts = vec_texts_reduced[train_len+val_len:len(vec_summaries_reduced)]
test_summaries = vec_summaries_reduced[train_len+val_len:len(vec_summaries_reduced)]
```


```python
print train_len
```

    18293



```python
def transform_out(output_text):
    output_len = len(output_text)
    transformed_output = np.zeros([output_len],dtype=np.int32)
    for i in xrange(0,output_len):
        transformed_output[i] = vocab_limit.index(vec2word(output_text[i]))
    return transformed_output   
```


```python
#Some MORE hyperparameters and other stuffs

hidden_size = 250
learning_rate = 0.003
K = 5
vocab_len = len(vocab_limit)
training_iters = 1 
```


```python
import tensorflow as tf

#placeholders
tf_text = tf.placeholder(tf.float32, [None,word_vec_dim])
tf_seq_len = tf.placeholder(tf.int32)
tf_summary = tf.placeholder(tf.int32,[None])
tf_output_len = tf.placeholder(tf.int32)
```


```python
def forward_encoder(inp,hidden,Wxh,Whh,Wattention,B,seq_len,inp_dim):

    Wattention = tf.nn.softmax(Wattention,0)
    hidden_forward = tf.TensorArray(size=seq_len,dtype=tf.float32)
    
    hidden_residuals = tf.TensorArray(size=K,dynamic_size=True,dtype=tf.float32,clear_after_read=False)
    hidden_residuals = hidden_residuals.unstack(tf.zeros([K,hidden_size],dtype=tf.float32))
    
    i=0
    j=K
    
    def fn(hidden_residuals):
        
        return hidden_residuals
    
    def cond(i,j,hidden,hidden_forward,hidden_residuals):
        return i < seq_len
    
    def body(i,j,hidden,hidden_forward,hidden_residuals):
        
        x = tf.reshape(inp[i],[1,inp_dim])
        
        hidden_residuals_stack = hidden_residuals.stack()
        
        RRA = tf.reduce_sum(tf.multiply(hidden_residuals_stack[j-K:j],Wattention),0)
        RRA = tf.reshape(RRA,[1,hidden_size])
        
        hidden_next = tf.nn.elu(tf.matmul(x,Wxh) + tf.matmul(hidden,Whh) + B + RRA)
        
        hidden = hidden_next
        
        hidden_residuals = tf.cond(tf.equal(j,seq_len-1+K),
                                   lambda: hidden_residuals,
                                   lambda: hidden_residuals.write(j,tf.reshape(hidden,[hidden_size])))

        hidden_forward = hidden_forward.write(i,tf.reshape(hidden,[hidden_size]))
        
        return i+1,j+1,hidden,hidden_forward,hidden_residuals
    
    _,_,_,hidden_forward,hidden_residuals = tf.while_loop(cond,body,[i,j,hidden,hidden_forward,hidden_residuals])
    
    hidden_residuals.close().mark_used()
    
    return hidden_forward.stack()
        
```


```python
def backward_encoder(inp,hidden,Wxh,Whh,Wattention,B,seq_len,inp_dim):
    Wattention = tf.nn.softmax(Wattention,0)
    hidden_backward = tf.TensorArray(size=seq_len,dtype=tf.float32)
    
    hidden_residuals = tf.TensorArray(size=K,dynamic_size=True,dtype=tf.float32,clear_after_read=False)
    hidden_residuals = hidden_residuals.unstack(tf.zeros([K,hidden_size],dtype=tf.float32))
    
    i=seq_len-1
    j=K
    
    def cond(i,j,hidden,hidden_backward,hidden_residuals):
        return i > -1
    
    def body(i,j,hidden,hidden_backward,hidden_residuals):
        
        x = tf.reshape(inp[i],[1,inp_dim])
        
        hidden_residuals_stack = hidden_residuals.stack()
        
        RRA = tf.reduce_sum(tf.multiply(hidden_residuals_stack[j-K:j],Wattention),0)
        RRA = tf.reshape(RRA,[1,hidden_size])
        
        hidden_next = tf.nn.elu(tf.matmul(x,Wxh) + tf.matmul(hidden,Whh) + B + RRA)
        
        hidden = hidden_next
        hidden_residuals = tf.cond(tf.equal(j,seq_len-1+K),
                                   lambda: hidden_residuals,
                                   lambda: hidden_residuals.write(j,tf.reshape(hidden,[hidden_size])))
        
        hidden_backward = hidden_backward.write(i,tf.reshape(hidden,[hidden_size]))
        
        return i-1,j+1,hidden,hidden_backward,hidden_residuals
    
    _,_,_,hidden_backward,hidden_residuals = tf.while_loop(cond,body,[i,j,hidden,hidden_backward,hidden_residuals])

    hidden_residuals.close().mark_used()
    
    return hidden_backward.stack()
        
```


```python
def decoder(inp,hidden,Wxh,Whh,B,RRA):
    hidden_next = tf.nn.elu(tf.matmul(inp,Wxh) + tf.matmul(hidden,Whh) + B + RRA)
    
    return hidden_next
```


```python
def score(hs,ht,Wa,seq_len):
    return tf.reshape(tf.matmul(tf.matmul(hs,Wa),tf.transpose(ht)),[seq_len])

def align(hs,ht,Wp,Vp,Wa,tf_seq_len):
   
    pd = tf.TensorArray(size=(2*D+1),dtype=tf.float32)
    
    sequence_length = tf_seq_len-tf.constant((2*D+1),dtype=tf.int32)
    sequence_length = tf.cast(sequence_length,dtype=tf.float32)
    
    pt_float = tf.multiply(sequence_length,tf.nn.sigmoid(tf.matmul(tf.tanh(tf.matmul(ht,Wp)),Vp)))
    pt_float = tf.add(pt_float,tf.cast(D,tf.float32))
    
    pt_float = tf.reshape(pt_float,[])
    
    pt = tf.cast(pt_float,tf.int32)
    
    sigma = tf.constant(D/2,dtype=tf.float32)
    
    i = 0
    pos = pt - D
    
    def cond(i,pos,pd):
        
        return i < (2*D+1)
                      
    def body(i,pos,pd):
        
        comp_1 = tf.cast(tf.square(tf.cast(pos,tf.float32)-pt_float),tf.float32)
        comp_2 = tf.cast(2*tf.square(sigma),tf.float32)
            
        pd = pd.write(i,tf.exp(-(comp_1/comp_2)))
            
        return i+1,pos+1,pd
                      
    i,pos,pd = tf.while_loop(cond,body,[i,pos,pd])
    
    local_hs = hs[(pt-D):(pt+D+1)]
    
    normalized_scores = tf.nn.softmax(score(local_hs,ht,Wa,2*D+1))
    
    pd=pd.stack()
    
    G = tf.multiply(normalized_scores,pd)
    G = tf.reshape(G,[2*D+1,1])
    
    return G,pt

```


```python
def model(tf_text,tf_seq_len,tf_output_len):
    
    #PARAMETERS
    
    #1.1 FORWARD ENCODER PARAMETERS
    
    initial_hidden_f = tf.zeros([1,hidden_size],dtype=tf.float32)
    Wxh_f = tf.Variable(tf.truncated_normal(shape=[word_vec_dim,hidden_size],stddev=0.01))
    Whh_f = tf.Variable(np.eye(hidden_size),dtype=tf.float32)
    B_f = tf.Variable(tf.zeros([1,hidden_size]),dtype=tf.float32)
    Wattention_f = tf.Variable(tf.zeros([K,1]),dtype=tf.float32)
                               
    #1.2 BACKWARD ENCODER PARAMETERS
    
    initial_hidden_b = tf.zeros([1,hidden_size],dtype=tf.float32)
    Wxh_b = tf.Variable(tf.truncated_normal(shape=[word_vec_dim,hidden_size],stddev=0.01))
    Whh_b = tf.Variable(np.eye(hidden_size),dtype=tf.float32)
    B_b = tf.Variable(tf.zeros([1,hidden_size]),dtype=tf.float32)
    Wattention_b = tf.Variable(tf.zeros([K,1]),dtype=tf.float32)
    
    #2 ATTENTION PARAMETERS
    
    Wp = tf.Variable(tf.truncated_normal(shape=[2*hidden_size,50],stddev=0.01))
    Vp = tf.Variable(tf.truncated_normal(shape=[50,1],stddev=0.01))
    Wa = tf.Variable(tf.truncated_normal(shape=[2*hidden_size,2*hidden_size],stddev=0.01))
    Wc = tf.Variable(tf.truncated_normal(shape=[4*hidden_size,2*hidden_size],stddev=0.01))
    
    #3 DECODER PARAMETERS
    
    Ws = tf.Variable(tf.truncated_normal(shape=[2*hidden_size,vocab_len],stddev=0.01))
    
    Wxh_d = tf.Variable(tf.truncated_normal(shape=[vocab_len,2*hidden_size],stddev=0.01))
    Whh_d = tf.Variable(np.eye(2*hidden_size),dtype=tf.float32)
    B_d = tf.Variable(tf.zeros([1,2*hidden_size]),dtype=tf.float32)
    
    hidden_residuals_d = tf.TensorArray(size=K,dynamic_size=True,dtype=tf.float32,clear_after_read=False)
    hidden_residuals_d = hidden_residuals_d.unstack(tf.zeros([K,2*hidden_size],dtype=tf.float32))
    
    Wattention_d = tf.Variable(tf.zeros([K,1]),dtype=tf.float32)
    
    output = tf.TensorArray(size=tf_output_len,dtype=tf.float32)
                               
    #BI-DIRECTIONAL ENCODER
                               
    hidden_forward = forward_encoder(tf_text,
                                     initial_hidden_f,
                                     Wxh_f,Whh_f,Wattention_f,B_f,
                                     tf_seq_len,
                                     word_vec_dim)
    
    hidden_backward = backward_encoder(tf_text,
                                       initial_hidden_b,
                                       Wxh_b,Whh_b,Wattention_b,B_b,
                                       tf_seq_len,
                                       word_vec_dim)
    
    encoded_hidden = tf.concat([hidden_forward,hidden_backward],1)
    
    #ATTENTION MECHANISM AND DECODER
    
    decoded_hidden = encoded_hidden[0]
    decoded_hidden = tf.reshape(decoded_hidden,[1,2*hidden_size])
    Wattention_d_normalized = tf.nn.softmax(Wattention_d)
                               
    i=0
    j=K
    
    def attention_decoder_cond(i,j,decoded_hidden,hidden_residuals_d,output):
        return i < tf_output_len
    
    def attention_decoder_body(i,j,decoded_hidden,hidden_residuals_d,output):
        
        #LOCAL ATTENTION
        
        G,pt = align(encoded_hidden,decoded_hidden,Wp,Vp,Wa,tf_seq_len)
        local_encoded_hidden = encoded_hidden[pt-D:pt+D+1]
        weighted_encoded_hidden = tf.multiply(local_encoded_hidden,G)
        context_vector = tf.reduce_sum(weighted_encoded_hidden,0)
        context_vector = tf.reshape(context_vector,[1,2*hidden_size])
        
        attended_hidden = tf.tanh(tf.matmul(tf.concat([context_vector,decoded_hidden],1),Wc))
        
        #DECODER
        
        y = tf.matmul(attended_hidden,Ws)
        
        output = output.write(i,tf.reshape(y,[vocab_len]))
        
        y = tf.nn.softmax(y)
        
        hidden_residuals_stack = hidden_residuals_d.stack()
        
        RRA = tf.reduce_sum(tf.multiply(hidden_residuals_stack[j-K:j],Wattention_d_normalized),0)
        RRA = tf.reshape(RRA,[1,2*hidden_size])
        
        decoded_hidden_next = decoder(y,attended_hidden,Wxh_d,Whh_d,B_d,RRA)
        
        decoded_hidden = decoded_hidden_next
        
        hidden_residuals_d = tf.cond(tf.equal(j,tf_output_len-1+K),
                                   lambda: hidden_residuals_d,
                                   lambda: hidden_residuals_d.write(j,tf.reshape(decoded_hidden,[2*hidden_size])))
        
        return i+1,j+1,decoded_hidden,hidden_residuals_d,output
    
    i,j,decoded_hidden,hidden_residuals_d,output = tf.while_loop(attention_decoder_cond,
                                            attention_decoder_body,
                                            [i,j,decoded_hidden,hidden_residuals_d,output])
    hidden_residuals_d.close().mark_used()
    
    output = output.stack()
    
    return output
```


```python
output = model(tf_text,tf_seq_len,tf_output_len)

#OPTIMIZER

cost = tf.reduce_mean(tf.nn.sparse_softmax_cross_entropy_with_logits(logits=output, labels=tf_summary))
optimizer = tf.train.AdamOptimizer(learning_rate=learning_rate).minimize(cost)

#PREDICTION

pred = tf.TensorArray(size=tf_output_len,dtype=tf.int32)

i=0

def cond_pred(i,pred):
    return i<tf_output_len
def body_pred(i,pred):
    pred = pred.write(i,tf.cast(tf.argmax(output[i]),tf.int32))
    return i+1,pred

i,pred = tf.while_loop(cond_pred,body_pred,[i,pred]) 

prediction = pred.stack()
```


```python
import string
from __future__ import print_function

init = tf.global_variables_initializer()


with tf.Session() as sess: # Start Tensorflow Session
    
    saver = tf.train.Saver() 
    # Prepares variable for saving the model
    sess.run(init) #initialize all variables
    step = 0   
    loss_list=[]
    acc_list=[]
    val_loss_list=[]
    val_acc_list=[]
    best_val_acc=0
    display_step = 1
    
    while step < training_iters:
        
        total_loss=0
        total_acc=0
        total_val_loss = 0
        total_val_acc = 0
           
        for i in xrange(0,train_len):
            
            train_out = transform_out(train_summaries[i][0:len(train_summaries[i])-1])
            
            if i%display_step==0:
                print("\nIteration: "+str(i))
                print("Training input sequence length: "+str(len(train_texts[i])))
                print("Training target outputs sequence length: "+str(len(train_out)))
            
                print("\nTEXT:")
                flag = 0
                for vec in train_texts[i]:
                    if vec2word(vec) in string.punctuation or flag==0:
                        print(str(vec2word(vec)),end='')
                    else:
                        print((" "+str(vec2word(vec))),end='')
                    flag=1

                print("\n")


            # Run optimization operation (backpropagation)
            _,loss,pred = sess.run([optimizer,cost,prediction],feed_dict={tf_text: train_texts[i], 
                                                    tf_seq_len: len(train_texts[i]), 
                                                    tf_summary: train_out,
                                                    tf_output_len: len(train_out)})
            
         
            if i%display_step==0:
                print("\nPREDICTED SUMMARY:\n")
                flag = 0
                for index in pred:
                    #if int(index)!=vocab_limit.index('eos'):
                    if vocab_limit[int(index)] in string.punctuation or flag==0:
                        print(str(vocab_limit[int(index)]),end='')
                    else:
                        print(" "+str(vocab_limit[int(index)]),end='')
                    flag=1
                print("\n")
                
                print("ACTUAL SUMMARY:\n")
                flag = 0
                for vec in train_summaries[i]:
                    if vec2word(vec)!='eos':
                        if vec2word(vec) in string.punctuation or flag==0:
                            print(str(vec2word(vec)),end='')
                        else:
                            print((" "+str(vec2word(vec))),end='')
                    flag=1

                print("\n")
            
                #print(hs)
            
                print("loss="+str(loss))
            
            #print(h)
            #print(out)
            #print(ht_s)
            
        step=step+1
    
```

    
    Iteration: 0
    Training input sequence length: 51
    Training target outputs sequence length: 4
    
    TEXT:
    i have bought several of the vitality canned dog food products and have found them all to be of good quality. the product looks more like a stew than a processed meat and it smells better. my labrador is finicky and she appreciates this product better than most.
    
    
    PREDICTED SUMMARY:
    
    competition 1960 cecco cecco
    
    ACTUAL SUMMARY:
    
    good quality dog food
    
    loss=10.4954
    
    Iteration: 1
    Training input sequence length: 37
    Training target outputs sequence length: 3
    
    TEXT:
    product arrived labeled as jumbo salted peanuts ... the peanuts were actually small sized unsalted. not sure if this was an error or if the vendor intended to represent the product as `` jumbo ''.
    
    
    PREDICTED SUMMARY:
    
    its food 15.6
    
    ACTUAL SUMMARY:
    
    not as advertised
    
    loss=10.245
    
    Iteration: 2
    Training input sequence length: 46
    Training target outputs sequence length: 2
    
    TEXT:
    if you are looking for the secret ingredient in robitussin i believe i have found it. i got this in addition to the root beer extract i ordered( which was good) and made some cherry soda. the flavor is very medicinal.
    
    
    PREDICTED SUMMARY:
    
    full-bodied dog
    
    ACTUAL SUMMARY:
    
    cough medicine
    
    loss=10.5442
    
    Iteration: 3
    Training input sequence length: 32
    Training target outputs sequence length: 2
    
    TEXT:
    great taffy at a great price. there was a wide assortment of yummy taffy. delivery was very quick. if your a taffy lover, this is a deal.
    
    
    PREDICTED SUMMARY:
    
    not advertised
    
    ACTUAL SUMMARY:
    
    great taffy
    
    loss=10.4141
    
    Iteration: 4
    Training input sequence length: 30
    Training target outputs sequence length: 4
    
    TEXT:
    this taffy is so good. it is very soft and chewy. the flavors are amazing. i would definitely recommend you buying it. very satisfying!!
    
    
    PREDICTED SUMMARY:
    
    not advertised advertised advertised
    
    ACTUAL SUMMARY:
    
    wonderful, tasty taffy
    
    loss=10.05
    
    Iteration: 5
    Training input sequence length: 29
    Training target outputs sequence length: 2
    
    TEXT:
    right now i 'm mostly just sprouting this so my cats can eat the grass. they love it. i rotate it around with wheatgrass and rye too
    
    
    PREDICTED SUMMARY:
    
    not food
    
    ACTUAL SUMMARY:
    
    yay barley
    
    loss=10.4446
    
    Iteration: 6
    Training input sequence length: 29
    Training target outputs sequence length: 3
    
    TEXT:
    this is a very healthy dog food. good for their digestion. also good for small puppies. my dog eats her required amount at every feeding.
    
    
    PREDICTED SUMMARY:
    
    cough food food
    
    ACTUAL SUMMARY:
    
    healthy dog food
    
    loss=8.06794
    
    Iteration: 7
    Training input sequence length: 24
    Training target outputs sequence length: 4
    
    TEXT:
    the strawberry twizzlers are my guilty pleasure- yummy. six pounds will be around for a while with my son and i.
    
    
    PREDICTED SUMMARY:
    
    wonderful food food food
    
    ACTUAL SUMMARY:
    
    strawberry twizzlers- yummy
    
    loss=10.3926
    
    Iteration: 8
    Training input sequence length: 45
    Training target outputs sequence length: 2
    
    TEXT:
    i love eating them and they are good for watching tv and looking at movies! it is not too sweet. i like to transfer them to a zip lock baggie so they stay fresh so i can take my time eating them.
    
    
    PREDICTED SUMMARY:
    
    wonderful food
    
    ACTUAL SUMMARY:
    
    poor taste
    
    loss=10.4998
    
    Iteration: 9
    Training input sequence length: 28
    Training target outputs sequence length: 3
    
    TEXT:
    i am very satisfied with my unk purchase. i shared these with others and we have all enjoyed them. i will definitely be ordering more.
    
    
    PREDICTED SUMMARY:
    
    wonderful food food
    
    ACTUAL SUMMARY:
    
    love it!
    
    loss=10.5893
    
    Iteration: 10
    Training input sequence length: 31
    Training target outputs sequence length: 3
    
    TEXT:
    candy was delivered very fast and was purchased at a reasonable price. i was home bound and unable to get to a store so this was perfect for me.
    
    
    PREDICTED SUMMARY:
    
    wonderful food food
    
    ACTUAL SUMMARY:
    
    home delivered unk
    
    loss=10.6324
    
    Iteration: 11
    Training input sequence length: 52
    Training target outputs sequence length: 2
    
    TEXT:
    my husband is a twizzlers addict. we 've bought these many times from amazon because we 're government employees living overseas and ca n't get them in the country we are assigned to. they 've always been fresh and tasty, packed well and arrive in a timely manner.
    
    
    PREDICTED SUMMARY:
    
    wonderful food
    
    ACTUAL SUMMARY:
    
    always fresh
    
    loss=10.8797
    
    Iteration: 12
    Training input sequence length: 68
    Training target outputs sequence length: 1
    
    TEXT:
    i bought these for my husband who is currently overseas. he loves these, and apparently his staff likes them unk< br/> there are generous amounts of twizzlers in each 16-ounce bag, and this was well worth the price.< a unk '' http: unk ''> twizzlers, strawberry, 16-ounce bags( pack of 6)< unk>
    
    
    PREDICTED SUMMARY:
    
    wonderful
    
    ACTUAL SUMMARY:
    
    twizzlers
    
    loss=8.25381
    
    Iteration: 13
    Training input sequence length: 31
    Training target outputs sequence length: 3
    
    TEXT:
    i can remember buying this candy as a kid and the quality has n't dropped in all these years. still a superb product you wo n't be disappointed with.
    
    
    PREDICTED SUMMARY:
    
    wonderful food food
    
    ACTUAL SUMMARY:
    
    delicious product!
    
    loss=9.78395
    
    Iteration: 14
    Training input sequence length: 21
    Training target outputs sequence length: 1
    
    TEXT:
    i love this candy. after weight watchers i had to cut back but still have a craving for it.
    
    
    PREDICTED SUMMARY:
    
    wonderful
    
    ACTUAL SUMMARY:
    
    twizzlers
    
    loss=5.16462
    
    Iteration: 15
    Training input sequence length: 72
    Training target outputs sequence length: 7
    
    TEXT:
    i have lived out of the us for over 7 yrs now, and i so miss my twizzlers!! when i go back to visit or someone visits me, i always stock up. all i can say is yum!< br/> sell these in mexico and you will have a faithful buyer, more often than i 'm able to buy them right now.
    
    
    PREDICTED SUMMARY:
    
    wonderful food food food food food food
    
    ACTUAL SUMMARY:
    
    please sell these in mexico!!
    
    loss=10.1993
    
    Iteration: 16
    Training input sequence length: 36
    Training target outputs sequence length: 3
    
    TEXT:
    product received is as unk< br/>< br/>< a unk '' http: unk ''> twizzlers, strawberry, 16-ounce bags( pack of 6)< unk>
    
    
    PREDICTED SUMMARY:
    
    wonderful taffy taffy
    
    ACTUAL SUMMARY:
    
    twizzlers- strawberry
    
    loss=4.43463
    
    Iteration: 17
    Training input sequence length: 43
    Training target outputs sequence length: 5
    
    TEXT:
    i was so glad amazon carried these batteries. i have a hard time finding them elsewhere because they are such a unique size. i need them for my garage door unk< br/> great deal for the price.
    
    
    PREDICTED SUMMARY:
    
    twizzlers taffy taffy taffy taffy
    
    ACTUAL SUMMARY:
    
    great bargain for the price
    
    loss=11.2824
    
    Iteration: 18
    Training input sequence length: 26
    Training target outputs sequence length: 5
    
    TEXT:
    this offer is a great price and a great taste, thanks amazon for selling this unk< br/>< br/> unk
    
    
    PREDICTED SUMMARY:
    
    twizzlers as as as as
    
    ACTUAL SUMMARY:
    
    this is my taste ...
    
    loss=10.4893
    
    Iteration: 19
    Training input sequence length: 60
    Training target outputs sequence length: 7
    
    TEXT:
    for those of us with celiac disease this product is a lifesaver and what could be better than getting it at almost half the price of the grocery or health food store! i love mccann 's instant oatmeal- all flavors!!!< br/>< br/> thanks,< br/> abby
    
    
    PREDICTED SUMMARY:
    
    twizzlers!!!!!!
    
    ACTUAL SUMMARY:
    
    love gluten free oatmeal!!!
    
    loss=7.05802
    
    Iteration: 20
    Training input sequence length: 59
    Training target outputs sequence length: 3
    
    TEXT:
    what else do you need to know? oatmeal, instant( make it with a half cup of low-fat milk and add raisins; nuke for 90 seconds). more expensive than kroger store brand oatmeal and maybe a little tastier or better texture or something. it 's still just oatmeal. mmm, convenient!
    
    
    PREDICTED SUMMARY:
    
    twizzlers!!
    
    ACTUAL SUMMARY:
    
    it 's oatmeal
    
    loss=11.5547
    
    Iteration: 21
    Training input sequence length: 79
    Training target outputs sequence length: 4
    
    TEXT:
    i ordered this for my wife as it was unk by our daughter. she has this almost every morning and likes all flavors. she 's happy, i 'm happy!!!< br/>< a unk '' http: unk ''> mccann 's instant irish oatmeal, variety pack of regular, apples& cinnamon, and maple& brown sugar, 10-count boxes( pack of 6)< unk>
    
    
    PREDICTED SUMMARY:
    
    twizzlers!!!
    
    ACTUAL SUMMARY:
    
    wife 's favorite breakfast
    
    loss=14.5898
    
    Iteration: 22
    Training input sequence length: 38
    Training target outputs sequence length: 1
    
    TEXT:
    i have mccann 's oatmeal every morning and by ordering it from amazon i am able to save almost$ 3.00 per unk< br/> it is a great product. tastes great and very healthy
    
    
    PREDICTED SUMMARY:
    
    twizzlers
    
    ACTUAL SUMMARY:
    
    unk
    
    loss=7.63945
    
    Iteration: 23
    Training input sequence length: 41
    Training target outputs sequence length: 3
    
    TEXT:
    mccann 's oatmeal is a good quality choice. our favorite is the apples and cinnamon, but we find that none of these are overly sugary. for a good hot breakfast in 2 minutes, this is excellent.
    
    
    PREDICTED SUMMARY:
    
    twizzlers!!
    
    ACTUAL SUMMARY:
    
    good hot breakfast
    
    loss=11.1365
    
    Iteration: 24
    Training input sequence length: 55
    Training target outputs sequence length: 4
    
    TEXT:
    we really like the mccann 's steel cut oats but find we do n't cook it up too unk< br/> this tastes much better to me than the grocery store brands and is just as unk< br/> anything that keeps me eating oatmeal regularly is a good thing.
    
    
    PREDICTED SUMMARY:
    
    twizzlers!!!
    
    ACTUAL SUMMARY:
    
    great taste and convenience
    
    loss=8.83746
    
    Iteration: 25
    Training input sequence length: 46
    Training target outputs sequence length: 2
    
    TEXT:
    this seems a little more wholesome than some of the supermarket brands, but it is somewhat mushy and does n't have quite as much flavor either. it did n't pass muster with my kids, so i probably wo n't buy it again.
    
    
    PREDICTED SUMMARY:
    
    twizzlers!
    
    ACTUAL SUMMARY:
    
    hearty oatmeal
    
    loss=12.347
    
    Iteration: 26
    Training input sequence length: 52
    Training target outputs sequence length: 1
    
    TEXT:
    good oatmeal. i like the apple cinnamon the best. though i would n't follow the directions on the package since it always comes out too soupy for my taste. that could just be me since i like my oatmeal really thick to add some milk on top of.
    
    
    PREDICTED SUMMARY:
    
    twizzlers
    
    ACTUAL SUMMARY:
    
    good
    
    loss=3.61977
    
    Iteration: 27
    Training input sequence length: 25
    Training target outputs sequence length: 1
    
    TEXT:
    the flavors are good. however, i do not see any unk between this and unk oats brand- they are both mushy.
    
    
    PREDICTED SUMMARY:
    
    twizzlers
    
    ACTUAL SUMMARY:
    
    mushy
    
    loss=16.738
    
    Iteration: 28
    Training input sequence length: 41
    Training target outputs sequence length: 2
    
    TEXT:
    this is the same stuff you can buy at the big box stores. there is nothing healthy about it. it is just carbs and sugars. save your money and get something that at least has some taste.
    
    
    PREDICTED SUMMARY:
    
    twizzlers taste
    
    ACTUAL SUMMARY:
    
    same stuff
    
    loss=15.9183
    
    Iteration: 29
    Training input sequence length: 25
    Training target outputs sequence length: 4
    
    TEXT:
    this oatmeal is not good. its mushy, soft, i do n't like it. quaker oats is the way to go.
    
    
    PREDICTED SUMMARY:
    
    good taste taste taste
    
    ACTUAL SUMMARY:
    
    do n't like it
    
    loss=12.6183
    
    Iteration: 30
    Training input sequence length: 37
    Training target outputs sequence length: 3
    
    TEXT:
    we 're used to spicy foods down here in south texas and these are not at all spicy. doubt very much habanero is used at all. could take it up a notch or two.
    
    
    PREDICTED SUMMARY:
    
    good taste taste
    
    ACTUAL SUMMARY:
    
    not ass kickin
    
    loss=11.6902
    
    Iteration: 31
    Training input sequence length: 80
    Training target outputs sequence length: 5
    
    TEXT:
    i roast at home with a unk popcorn popper( but i do it outside, of course). these beans( coffee bean direct green mexican altura) seem to be well-suited for this method. the first and second cracks are distinct, and i 've roasted the beans from medium to slightly dark with great results every time. the aroma is strong and persistent. the taste is smooth, velvety, yet


```python

```