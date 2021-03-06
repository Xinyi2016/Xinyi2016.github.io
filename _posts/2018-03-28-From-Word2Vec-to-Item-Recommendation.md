---
layout: post
title: From Word2Vec to Item Recommendation
author: sara
categories: [Machine Learning]
image: assets/images/dogs/leisure-wildlife-photography-pet-photography-dog-159557.jpeg
featured: False
---

As a data analyst, I'm always looking for ways to leverage massive dataset to automate customer-facing experiences. Recently I'm focusing on using word2vec to power a recommendation engine.

Following is a minimalist script explaining how to apply Word2Vec for item recommendation and the data science behind it. 

The demo data can be downloaded here: https://github.com/zygmuntz/goodbooks-10k

## Step 1. Make every user's to-read list as a sentence and feed it into word2vec

Given a dataset large enough, it can find out similar books


```python
import collections
import math
import random
import numpy as np
import tensorflow as tf
import pandas as pd
from six.moves import xrange
```

It's worth mentioning that the syntax of `xrange` and `range` are the same.

* xrange(stop)
* xrange(start, stop[, step])

The major difference is that `range` outputs tuples while `xrange` gives us a generator, which is more efficient when data are too large to fit in memory.


```python
tr = pd.read_csv( 'to_read.csv' )
b = pd.read_csv( 'books.csv' )
bt = tr.merge( b[[ 'book_id', 'title']], on = 'book_id' )
```


```python
bt.sample(10)
```




<div>
<style>
    .dataframe thead tr:only-child th {
        text-align: right;
    }

    .dataframe thead th {
        text-align: left;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>user_id</th>
      <th>book_id</th>
      <th>title</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>535344</th>
      <td>12001</td>
      <td>1587</td>
      <td>Saving CeeCee Honeycutt</td>
    </tr>
    <tr>
      <th>127117</th>
      <td>996</td>
      <td>4055</td>
      <td>Jesus' Son</td>
    </tr>
    <tr>
      <th>57970</th>
      <td>43087</td>
      <td>457</td>
      <td>The Historian</td>
    </tr>
    <tr>
      <th>197783</th>
      <td>14942</td>
      <td>148</td>
      <td>Girl with a Pearl Earring</td>
    </tr>
    <tr>
      <th>754480</th>
      <td>40397</td>
      <td>359</td>
      <td>And the Mountains Echoed</td>
    </tr>
    <tr>
      <th>105202</th>
      <td>9323</td>
      <td>187</td>
      <td>Uglies (Uglies, #1)</td>
    </tr>
    <tr>
      <th>692645</th>
      <td>40233</td>
      <td>3727</td>
      <td>The Rook (The Checquy Files, #1)</td>
    </tr>
    <tr>
      <th>905310</th>
      <td>42392</td>
      <td>1028</td>
      <td>Truly Madly Guilty</td>
    </tr>
    <tr>
      <th>728410</th>
      <td>2452</td>
      <td>4402</td>
      <td>The Perfect Hope (Inn Boonsboro, #3)</td>
    </tr>
    <tr>
      <th>100382</th>
      <td>15170</td>
      <td>4</td>
      <td>To Kill a Mockingbird</td>
    </tr>
  </tbody>
</table>
</div>



Detailed exploration and data cleaning process -> 


```python
raw = bt.groupby('user_id')['title'].apply(list)
assert len(raw)==tr.user_id.nunique()
```


```python
words = [b for books in raw.values for b in books]
n_words = bt.title.nunique()
print("There are {} books and {} to-read records".format(n_words,len(words)))
```

    There are 9951 books and 912705 to-read records



```python
def build_dataset(words, n_words):
    count = []
    count.extend(collections.Counter(words).most_common(n_words))
    dictionary = dict()
    for word, _ in count:
        dictionary[word] = len(dictionary)
    data = list()
    for word in words:
        index = dictionary.get(word, 0)
        data.append(index)
    reversed_dictionary = dict(zip(dictionary.values(), dictionary.keys()))
    return data, count, dictionary, reversed_dictionary


```


```python
data, count, dictionary, reverse_dictionary = build_dataset(words,n_words)
```


```python
words[0]
```




    "The Aviator's Wife"



## Step 2. Build a skip-gram model

We can create our own word embeddings easily using `gensim`, an open source Python library focusing on topic modeling.

Here I will have a try on `TensorFlow`. 


```python
data_index = 0

def generate_batch(batch_size, num_skips, skip_window):
    global data_index
    assert batch_size % num_skips == 0
    assert num_skips <= 2*skip_window
    batch = np.ndarray(shape=(batch_size), dtype=np.int32)
    labels = np.ndarray(shape=(batch_size, 1), dtype=np.int32)
    span = 2 * skip_window + 1  # [ skip_window target skip_window ]
    buffer = collections.deque(maxlen=span)
    if data_index + span > len(data):
        data_index = 0
    buffer.extend(data[data_index:data_index + span])
    data_index += span
    for i in range(batch_size // num_skips):
        context_words = [w for w in range(span) if w != skip_window]
        words_to_use = random.sample(context_words, num_skips)
        for j, context_word in enumerate(words_to_use):
            batch[i * num_skips + j] = buffer[skip_window]
            labels[i * num_skips + j, 0] = buffer[context_word]
        if data_index == len(data):
            for book in data[:span]:
                buffer.append(book)
            data_index = span
        else:
            buffer.append(data[data_index])
            data_index += 1
  # Backtrack a little bit to avoid skipping words in the end of a batch
    data_index = (data_index + len(data) - span) % len(data)
    return batch, labels

batch, labels = generate_batch(batch_size=8, num_skips=2, skip_window=1)
for i in range(8):
  print(batch[i], reverse_dictionary[batch[i]],
        '->', labels[i, 0], reverse_dictionary[labels[i, 0]])


```

    86 Me Before You (Me Before You, #1) -> 720 The Aviator's Wife
    86 Me Before You (Me Before You, #1) -> 380 The Husband's Secret
    380 The Husband's Secret -> 86 Me Before You (Me Before You, #1)
    380 The Husband's Secret -> 175 A Little Life
    175 A Little Life -> 380 The Husband's Secret
    175 A Little Life -> 51 Go Set a Watchman
    51 Go Set a Watchman -> 175 A Little Life
    51 Go Set a Watchman -> 1095 The Marriage of Opposites



```python
batch_size = 128
embedding_size = 128  # Dimension of the embedding vector.
skip_window = 1       # How many words to consider left and right.
num_skips = 2         # How many times to reuse an input to generate a label.
num_sampled = 64      # Number of negative examples to sample.

# We pick a random validation set to sample nearest neighbors. Here we limit the
# validation samples to the words that have a low numeric ID, which by
# construction are also the most frequent. These 3 variables are used only for
# displaying model accuracy, they don't affect calculation.
valid_size = 16     # Random set of words to evaluate similarity on.
valid_window = 100  # Only pick dev samples in the head of the distribution.
valid_examples = np.random.choice(valid_window, valid_size, replace=False)


graph = tf.Graph()

with graph.as_default():

  # Input data.
    train_inputs = tf.placeholder(tf.int32, shape=[batch_size])
    train_labels = tf.placeholder(tf.int32, shape=[batch_size, 1])
    valid_dataset = tf.constant(valid_examples, dtype=tf.int32)

  # Ops and variables pinned to the CPU because of missing GPU implementation
    with tf.device('/cpu:0'):
        embeddings = tf.Variable(
            tf.random_uniform([n_words,embedding_size],-1.0,1.0))
        embed = tf.nn.embedding_lookup(embeddings, train_inputs)

        # Construct the variables for the NCE loss
        nce_weights = tf.Variable(
            tf.truncated_normal([n_words, embedding_size],
                                stddev=1.0 / math.sqrt(embedding_size)))
        nce_biases = tf.Variable(tf.zeros([n_words]))

    # Compute the average NCE loss for the batch.
    # tf.nce_loss automatically draws a new sample of the negative labels each
    # time we evaluate the loss.
    loss = tf.reduce_mean(
      tf.nn.nce_loss(weights=nce_weights,
                     biases=nce_biases,
                     labels=train_labels,
                     inputs=embed,
                     num_sampled=num_sampled,
                     num_classes=n_words))

    # Construct the SGD optimizer using a learning rate of 1.0.
    optimizer = tf.train.GradientDescentOptimizer(1.0).minimize(loss)

    # Compute the cosine similarity between minibatch examples and all embeddings.
    norm = tf.sqrt(tf.reduce_sum(tf.square(embeddings), 1, keep_dims=True))
    normalized_embeddings = embeddings / norm
    valid_embeddings = tf.nn.embedding_lookup(
      normalized_embeddings, valid_dataset)
    similarity = tf.matmul(
      valid_embeddings, normalized_embeddings, transpose_b=True)

    # Add variable initializer.
    init = tf.global_variables_initializer()


```

## Step 3. Start Training


```python
num_steps = 100001

with tf.Session(graph=graph) as session:
    # We must initialize all variables before we use them.
    init.run()
    print('Initialized')

    average_loss = 0
    for step in xrange(num_steps):
        batch_inputs, batch_labels = generate_batch(
            batch_size, num_skips, skip_window)
        feed_dict = {train_inputs: batch_inputs, train_labels: batch_labels}

        # We perform one update step by evaluating the optimizer op (including it
        # in the list of returned values for session.run()
        _, loss_val = session.run([optimizer, loss], feed_dict=feed_dict)
        average_loss += loss_val

        if step % 2000 == 0:
            if step > 0:
                average_loss /= 2000
          # The average loss is an estimate of the loss over the last 2000 batches.
            print('Average loss at step ', step, ': ', average_loss)
            average_loss = 0

    # Note that this is expensive (~20% slowdown if computed every 500 steps)
        if step % 10000 == 0:
            sim = similarity.eval()
            for i in xrange(valid_size):
                valid_word = reverse_dictionary[valid_examples[i]]
                top_k = 3  # number of nearest neighbors
                nearest = (-sim[i, :]).argsort()[1:top_k + 1]
                log_str = 'Nearest to %s:' % valid_word
                for k in xrange(top_k):
                    close_word = reverse_dictionary[nearest[k]]
                    log_str = '%s %s,' % (log_str, close_word)
                print(log_str)
    final_embeddings = normalized_embeddings.eval()

```

    Initialized
    Average loss at step  0 :  201.053588867
    Nearest to The Hitchhiker's Guide to the Galaxy (Hitchhiker's Guide to the Galaxy, #1): Batman: The Dark Knight Returns (The Dark Knight Saga, #1), Selected Stories, If I Stay (If I Stay, #1),
    Nearest to Cinder (The Lunar Chronicles, #1): Eleven Minutes, The Five Love Languages of Children, Austenland (Austenland, #1),
    Nearest to The Kite Runner: The Rumor, Labyrinth (Languedoc, #1), The Sea of Trolls (Sea of Trolls, #1),
    Nearest to The Glass Castle: The Witch's Daughter (The Witch's Daughter, #1), The Love Dare, Holding Up the Universe,
    Nearest to Extremely Loud and Incredibly Close: Not My Daughter, The Buried Giant, The Andromeda Strain,
    Nearest to Love in the Time of Cholera: The Woodlanders, The Ringworld Engineers (Ringworld #2), One Fish, Two Fish, Red Fish, Blue Fish,
    Nearest to The Catcher in the Rye: Ancillary Sword (Imperial Radch, #2), I Shall Wear Midnight (Discworld, #38; Tiffany Aching, #4), The Sands of Time,
    Nearest to Crime and Punishment: Dairy Queen (Dairy Queen, #1), Among Others, Millennium Snow, Vol. 1,
    Nearest to Animal Farm: How to Save a Life, Officer Buckle & Gloria, The Perfect Storm: A True Story of Men Against the Sea,
    Nearest to Wicked: The Life and Times of the Wicked Witch of the West (The Wicked Years, #1): My Life, Salem Falls, The Ironwood Tree (The Spiderwick Chronicles, #4),
    Nearest to The Perks of Being a Wallflower: The Dragonslayer (Bone, #4), Deliverance, Rendezvous with Rama (Rama, #1),
    Nearest to War and Peace: East, The Elvenbane (Halfblood Chronicles, #1), Medium Raw: A Bloody Valentine to the World of Food and the People Who Cook,
    Nearest to Brave New World: On Mystic Lake, Fledgling, 77 Shadow Street (Pendleton, #1),
    Nearest to The Girl with the Dragon Tattoo (Millennium, #1): The Secret of the Unicorn (Tintin, #11), Player Piano, Angels Fall,
    Nearest to The Casual Vacancy: The Prayer of Jabez:  Breaking Through to the Blessed Life, The Weekenders, The Assassin's Blade (Throne of Glass, #0.1-0.5),
    Nearest to Lord of the Flies: Cloudy With a Chance of Meatballs, First Frost (Waverley Family, #2), Passing,
    Average loss at step  2000 :  50.8570902784
    Average loss at step  4000 :  8.48856670332
    Average loss at step  6000 :  5.50654278445
    Average loss at step  8000 :  4.89954230261
    Average loss at step  10000 :  4.66325466359
    Nearest to The Hitchhiker's Guide to the Galaxy (Hitchhiker's Guide to the Galaxy, #1): The Notebook (The Notebook, #1), Me Talk Pretty One Day, 20th Century Ghosts,
    Nearest to Cinder (The Lunar Chronicles, #1): The Five Love Languages of Children, Eleven Minutes, Bully (Fall Away, #1),
    Nearest to The Kite Runner: Tuesdays with Morrie, Catch-22, The Omnivore's Dilemma: A Natural History of Four Meals,
    Nearest to The Glass Castle: The Witch's Daughter (The Witch's Daughter, #1), In Cold Blood, Hot, Flat, and Crowded: Why We Need a Green Revolution--and How It Can Renew America,
    Nearest to Extremely Loud and Incredibly Close: The Poisonwood Bible, The Buried Giant, The Andromeda Strain,
    Nearest to Love in the Time of Cholera: One Fish, Two Fish, Red Fish, Blue Fish, Little Women (Little Women, #1), The Red Garden,
    Nearest to The Catcher in the Rye: Ancillary Sword (Imperial Radch, #2), The Personal MBA: Master the Art of Business, I Shall Wear Midnight (Discworld, #38; Tiffany Aching, #4),
    Nearest to Crime and Punishment: Dairy Queen (Dairy Queen, #1), Catch-22, The Great Gatsby,
    Nearest to Animal Farm: How to Save a Life, Infinite Jest, The Perfect Storm: A True Story of Men Against the Sea,
    Nearest to Wicked: The Life and Times of the Wicked Witch of the West (The Wicked Years, #1): My Life, Atonement, Orphan X (Orphan X, #1),
    Nearest to The Perks of Being a Wallflower: Catch-22, Rendezvous with Rama (Rama, #1), The Dragonslayer (Bone, #4),
    Nearest to War and Peace: Everything, Everything, Medium Raw: A Bloody Valentine to the World of Food and the People Who Cook, East,
    Nearest to Brave New World: Fledgling, Outlander (Outlander, #1), Tuesdays with Morrie,
    Nearest to The Girl with the Dragon Tattoo (Millennium, #1): The Secret of the Unicorn (Tintin, #11), Dewey: The Small-Town Library Cat Who Touched the World, Story of O (Story of O #1),
    Nearest to The Casual Vacancy: The Weekenders, The Prayer of Jabez:  Breaking Through to the Blessed Life, Life and Other Near-Death Experiences,
    Nearest to Lord of the Flies: Passing, Snow, A Short History of Nearly Everything,
    Average loss at step  12000 :  4.49367789495
    Average loss at step  14000 :  4.3219834007
    Average loss at step  16000 :  4.2673389703
    Average loss at step  18000 :  4.09554072869
    Average loss at step  20000 :  4.04103688669
    Nearest to The Hitchhiker's Guide to the Galaxy (Hitchhiker's Guide to the Galaxy, #1): The Notebook (The Notebook, #1), Me Talk Pretty One Day, The Iliad,
    Nearest to Cinder (The Lunar Chronicles, #1): The Five Love Languages of Children, Delirium (Delirium, #1), The Fault in Our Stars,
    Nearest to The Kite Runner: Tuesdays with Morrie, The Omnivore's Dilemma: A Natural History of Four Meals, The House of the Spirits,
    Nearest to The Glass Castle: In Cold Blood, The Witch's Daughter (The Witch's Daughter, #1), The Bonesetter's Daughter,
    Nearest to Extremely Loud and Incredibly Close: The Poisonwood Bible, The Widow of the South, The Memory Keeper's Daughter,
    Nearest to Love in the Time of Cholera: I Know Why the Caged Bird Sings, One Fish, Two Fish, Red Fish, Blue Fish, The Underground Railroad,
    Nearest to The Catcher in the Rye: Ancillary Sword (Imperial Radch, #2), Democracy in America , The Personal MBA: Master the Art of Business,
    Nearest to Crime and Punishment: Dairy Queen (Dairy Queen, #1), Zen and the Art of Motorcycle Maintenance: An Inquiry Into Values, The Great Gatsby,
    Nearest to Animal Farm: Infinite Jest, Julie and Julia: 365 Days, 524 Recipes, 1 Tiny Apartment Kitchen: How One Girl Risked Her Marriage, Her Job, and Her Sanity to Master the Art of Living, How to Save a Life,
    Nearest to Wicked: The Life and Times of the Wicked Witch of the West (The Wicked Years, #1): Atonement, I, Claudius (Claudius, #1), My Life,
    Nearest to The Perks of Being a Wallflower: Catch-22, Lolita, Rendezvous with Rama (Rama, #1),
    Nearest to War and Peace: Norwegian Wood, Everything, Everything, Medium Raw: A Bloody Valentine to the World of Food and the People Who Cook,
    Nearest to Brave New World: Tuesdays with Morrie, Blue Like Jazz: Nonreligious Thoughts on Christian Spirituality, Outlander (Outlander, #1),
    Nearest to The Girl with the Dragon Tattoo (Millennium, #1): The Secret of the Unicorn (Tintin, #11), Dewey: The Small-Town Library Cat Who Touched the World, The Housekeeper and the Professor,
    Nearest to The Casual Vacancy: The Fault in Our Stars, The Weekenders, The Prayer of Jabez:  Breaking Through to the Blessed Life,
    Nearest to Lord of the Flies: Passing, Snow, A Short History of Nearly Everything,
    Average loss at step  22000 :  3.99072129893
    Average loss at step  24000 :  3.90868891752
    Average loss at step  26000 :  3.8537153455
    Average loss at step  28000 :  3.77176816392
    Average loss at step  30000 :  3.82985007036
    Nearest to The Hitchhiker's Guide to the Galaxy (Hitchhiker's Guide to the Galaxy, #1): The Notebook (The Notebook, #1), Me Talk Pretty One Day, The Iliad,
    Nearest to Cinder (The Lunar Chronicles, #1): The Fault in Our Stars, Delirium (Delirium, #1), Ruby Red (Precious Stone Trilogy, #1),
    Nearest to The Kite Runner: Tuesdays with Morrie, The House of the Spirits, The Omnivore's Dilemma: A Natural History of Four Meals,
    Nearest to The Glass Castle: The Bonesetter's Daughter, The Other Boleyn Girl (The Plantagenet and Tudor Novels, #9), In Cold Blood,
    Nearest to Extremely Loud and Incredibly Close: The Poisonwood Bible, The Widow of the South, The Memory Keeper's Daughter,
    Nearest to Love in the Time of Cholera: I Know Why the Caged Bird Sings, Atlas Shrugged, The Underground Railroad,
    Nearest to The Catcher in the Rye: Democracy in America , Ancillary Sword (Imperial Radch, #2), Everything We Keep (Everything We Keep, #1),
    Nearest to Crime and Punishment: Zen and the Art of Motorcycle Maintenance: An Inquiry Into Values, Slaughterhouse-Five, The Great Gatsby,
    Nearest to Animal Farm: The Good Earth (House of Earth, #1), Infinite Jest, Julie and Julia: 365 Days, 524 Recipes, 1 Tiny Apartment Kitchen: How One Girl Risked Her Marriage, Her Job, and Her Sanity to Master the Art of Living,
    Nearest to Wicked: The Life and Times of the Wicked Witch of the West (The Wicked Years, #1): Atonement, I, Claudius (Claudius, #1), Ender's Game (Ender's Saga, #1),
    Nearest to The Perks of Being a Wallflower: Catch-22, To Kill a Mockingbird, The Garden of Eden,
    Nearest to War and Peace: Norwegian Wood, Everything, Everything, Jonathan Strange & Mr Norrell,
    Nearest to Brave New World: Outlander (Outlander, #1), Tuesdays with Morrie, Blue Like Jazz: Nonreligious Thoughts on Christian Spirituality,
    Nearest to The Girl with the Dragon Tattoo (Millennium, #1): The Secret of the Unicorn (Tintin, #11), Dewey: The Small-Town Library Cat Who Touched the World, The Housekeeper and the Professor,
    Nearest to The Casual Vacancy: The Fault in Our Stars, The Prayer of Jabez:  Breaking Through to the Blessed Life, Wild: From Lost to Found on the Pacific Crest Trail,
    Nearest to Lord of the Flies: Snow, Passing, A Short History of Nearly Everything,
    Average loss at step  32000 :  3.72429895306
    Average loss at step  34000 :  3.67616319287
    Average loss at step  36000 :  3.65157854366
    Average loss at step  38000 :  3.60568102384
    Average loss at step  40000 :  3.5629105919
    Nearest to The Hitchhiker's Guide to the Galaxy (Hitchhiker's Guide to the Galaxy, #1): The Notebook (The Notebook, #1), Me Talk Pretty One Day, A Streetcar Named Desire,
    Nearest to Cinder (The Lunar Chronicles, #1): The Fault in Our Stars, Embrace (The Violet Eden Chapters, #1), Seven Deadly Wonders (Jack West Jr, #1),
    Nearest to The Kite Runner: Tuesdays with Morrie, The Omnivore's Dilemma: A Natural History of Four Meals, The House of the Spirits,
    Nearest to The Glass Castle: The Bonesetter's Daughter, The Other Boleyn Girl (The Plantagenet and Tudor Novels, #9), Peony in Love,
    Nearest to Extremely Loud and Incredibly Close: The Poisonwood Bible, The Widow of the South, The Memory Keeper's Daughter,
    Nearest to Love in the Time of Cholera: I Know Why the Caged Bird Sings, Embrace the Night (Cassandra Palmer, #3), The Grapes of Wrath,
    Nearest to The Catcher in the Rye: Democracy in America , Ancillary Sword (Imperial Radch, #2), War and Peace,
    Nearest to Crime and Punishment: Zen and the Art of Motorcycle Maintenance: An Inquiry Into Values, Slaughterhouse-Five, To Have and Have Not,
    Nearest to Animal Farm: The Good Earth (House of Earth, #1), Infinite Jest, Julie and Julia: 365 Days, 524 Recipes, 1 Tiny Apartment Kitchen: How One Girl Risked Her Marriage, Her Job, and Her Sanity to Master the Art of Living,
    Nearest to Wicked: The Life and Times of the Wicked Witch of the West (The Wicked Years, #1): Ender's Game (Ender's Saga, #1), Atonement, I, Claudius (Claudius, #1),
    Nearest to The Perks of Being a Wallflower: Catch-22, To Kill a Mockingbird, The Garden of Eden,
    Nearest to War and Peace: Norwegian Wood, Jonathan Strange & Mr Norrell, Nine Stories,
    Nearest to Brave New World: Outlander (Outlander, #1), Blue Like Jazz: Nonreligious Thoughts on Christian Spirituality, Tuesdays with Morrie,
    Nearest to The Girl with the Dragon Tattoo (Millennium, #1): The Secret of the Unicorn (Tintin, #11), The Housekeeper and the Professor, Dewey: The Small-Town Library Cat Who Touched the World,
    Nearest to The Casual Vacancy: The Fault in Our Stars, The Prayer of Jabez:  Breaking Through to the Blessed Life, Wild: From Lost to Found on the Pacific Crest Trail,
    Nearest to Lord of the Flies: Snow, Harry Potter and the Deathly Hallows (Harry Potter, #7), A Short History of Nearly Everything,
    Average loss at step  42000 :  3.5407050733
    Average loss at step  44000 :  3.60401493472
    Average loss at step  46000 :  3.55708793867
    Average loss at step  48000 :  3.49812850416
    Average loss at step  50000 :  3.46880627728
    Nearest to The Hitchhiker's Guide to the Galaxy (Hitchhiker's Guide to the Galaxy, #1): The Notebook (The Notebook, #1), The Plot Against America, Me Talk Pretty One Day,
    Nearest to Cinder (The Lunar Chronicles, #1): The Fault in Our Stars, Seven Deadly Wonders (Jack West Jr, #1), Embrace (The Violet Eden Chapters, #1),
    Nearest to The Kite Runner: Tuesdays with Morrie, A Clockwork Orange, The Omnivore's Dilemma: A Natural History of Four Meals,
    Nearest to The Glass Castle: The Bonesetter's Daughter, Peony in Love, In Cold Blood,
    Nearest to Extremely Loud and Incredibly Close: The Poisonwood Bible, The Widow of the South, Lolita,
    Nearest to Love in the Time of Cholera: I Know Why the Caged Bird Sings, The Catcher in the Rye, The Grapes of Wrath,
    Nearest to The Catcher in the Rye: Democracy in America , Ancillary Sword (Imperial Radch, #2), Love in the Time of Cholera,
    Nearest to Crime and Punishment: Zen and the Art of Motorcycle Maintenance: An Inquiry Into Values, Slaughterhouse-Five, To Have and Have Not,
    Nearest to Animal Farm: The Good Earth (House of Earth, #1), Infinite Jest, Gone with the Wind,
    Nearest to Wicked: The Life and Times of the Wicked Witch of the West (The Wicked Years, #1): Ender's Game (Ender's Saga, #1), Atonement, I, Claudius (Claudius, #1),
    Nearest to The Perks of Being a Wallflower: Catch-22, To Kill a Mockingbird, The Garden of Eden,
    Nearest to War and Peace: Norwegian Wood, Nine Stories, The God Delusion,
    Nearest to Brave New World: The Giver (The Giver, #1), Tuesdays with Morrie, The Old Man and the Sea,
    Nearest to The Girl with the Dragon Tattoo (Millennium, #1): The Secret of the Unicorn (Tintin, #11), The Housekeeper and the Professor, Dewey: The Small-Town Library Cat Who Touched the World,
    Nearest to The Casual Vacancy: The Fault in Our Stars, Wild: From Lost to Found on the Pacific Crest Trail, The Prayer of Jabez:  Breaking Through to the Blessed Life,
    Nearest to Lord of the Flies: Snow, Harry Potter and the Deathly Hallows (Harry Potter, #7), The Kite Runner,
    Average loss at step  52000 :  3.43265928054
    Average loss at step  54000 :  3.42167698848
    Average loss at step  56000 :  3.39775447059
    Average loss at step  58000 :  3.47592809737
    Average loss at step  60000 :  3.47204799223
    Nearest to The Hitchhiker's Guide to the Galaxy (Hitchhiker's Guide to the Galaxy, #1): Middlesex, A Clockwork Orange, The Notebook (The Notebook, #1),
    Nearest to Cinder (The Lunar Chronicles, #1): The Fault in Our Stars, Embrace (The Violet Eden Chapters, #1), Seven Deadly Wonders (Jack West Jr, #1),
    Nearest to The Kite Runner: Tuesdays with Morrie, A Clockwork Orange, 1984,
    Nearest to The Glass Castle: The Bonesetter's Daughter, Peony in Love, In Cold Blood,
    Nearest to Extremely Loud and Incredibly Close: The Poisonwood Bible, The Widow of the South, Lolita,
    Nearest to Love in the Time of Cholera: I Know Why the Caged Bird Sings, The Divine Comedy, War and Peace,
    Nearest to The Catcher in the Rye: Ancillary Sword (Imperial Radch, #2), Love in the Time of Cholera, Democracy in America ,
    Nearest to Crime and Punishment: Slaughterhouse-Five, To Have and Have Not, Zen and the Art of Motorcycle Maintenance: An Inquiry Into Values,
    Nearest to Animal Farm: The Good Earth (House of Earth, #1), Gone with the Wind, Rabbit, Run (Rabbit Angstrom #1),
    Nearest to Wicked: The Life and Times of the Wicked Witch of the West (The Wicked Years, #1): Ender's Game (Ender's Saga, #1), Atonement, 1984,
    Nearest to The Perks of Being a Wallflower: Catch-22, To Kill a Mockingbird, The Golden Compass (His Dark Materials, #1),
    Nearest to War and Peace: Nine Stories, Jonathan Strange & Mr Norrell, Harry Potter and the Deathly Hallows (Harry Potter, #7),
    Nearest to Brave New World: Middlesex, The Giver (The Giver, #1), Catch-22,
    Nearest to The Girl with the Dragon Tattoo (Millennium, #1): The Secret of the Unicorn (Tintin, #11), The Housekeeper and the Professor, Dewey: The Small-Town Library Cat Who Touched the World,
    Nearest to The Casual Vacancy: The Fault in Our Stars, Wild: From Lost to Found on the Pacific Crest Trail, The Prayer of Jabez:  Breaking Through to the Blessed Life,
    Nearest to Lord of the Flies: Snow, The Kite Runner, Harry Potter and the Deathly Hallows (Harry Potter, #7),
    Average loss at step  62000 :  3.3924072181
    Average loss at step  64000 :  3.36924781489
    Average loss at step  66000 :  3.33055694771
    Average loss at step  68000 :  3.31733061743
    Average loss at step  70000 :  3.31788296509
    Nearest to The Hitchhiker's Guide to the Galaxy (Hitchhiker's Guide to the Galaxy, #1): A Clockwork Orange, Catch-22, Middlesex,
    Nearest to Cinder (The Lunar Chronicles, #1): Seven Deadly Wonders (Jack West Jr, #1), Embrace (The Violet Eden Chapters, #1), The Fault in Our Stars,
    Nearest to The Kite Runner: Tuesdays with Morrie, A Clockwork Orange, 1984,
    Nearest to The Glass Castle: Peony in Love, The Bonesetter's Daughter, The Other Boleyn Girl (The Plantagenet and Tudor Novels, #9),
    Nearest to Extremely Loud and Incredibly Close: The Poisonwood Bible, Lolita, The Omnivore's Dilemma: A Natural History of Four Meals,
    Nearest to Love in the Time of Cholera: The Catcher in the Rye, War and Peace, The Divine Comedy,
    Nearest to The Catcher in the Rye: Love in the Time of Cholera, Pride and Prejudice, Ancillary Sword (Imperial Radch, #2),
    Nearest to Crime and Punishment: Zen and the Art of Motorcycle Maintenance: An Inquiry Into Values, Slaughterhouse-Five, To Have and Have Not,
    Nearest to Animal Farm: The Good Earth (House of Earth, #1), Gone with the Wind, Infinite Jest,
    Nearest to Wicked: The Life and Times of the Wicked Witch of the West (The Wicked Years, #1): Ender's Game (Ender's Saga, #1), Atonement, The Hitchhiker's Guide to the Galaxy (Hitchhiker's Guide to the Galaxy, #1),
    Nearest to The Perks of Being a Wallflower: Catch-22, To Kill a Mockingbird, The Curious Incident of the Dog in the Night-Time,
    Nearest to War and Peace: Harry Potter and the Deathly Hallows (Harry Potter, #7), Jonathan Strange & Mr Norrell, The God Delusion,
    Nearest to Brave New World: Catch-22, Middlesex, To Kill a Mockingbird,
    Nearest to The Girl with the Dragon Tattoo (Millennium, #1): The Secret of the Unicorn (Tintin, #11), The Housekeeper and the Professor, Dewey: The Small-Town Library Cat Who Touched the World,
    Nearest to The Casual Vacancy: Wild: From Lost to Found on the Pacific Crest Trail, The Fault in Our Stars, The Prayer of Jabez:  Breaking Through to the Blessed Life,
    Nearest to Lord of the Flies: Snow, Harry Potter and the Deathly Hallows (Harry Potter, #7), The Kite Runner,
    Average loss at step  72000 :  3.37482465214
    Average loss at step  74000 :  3.42257392645
    Average loss at step  76000 :  3.31818997788
    Average loss at step  78000 :  3.29870229846
    Average loss at step  80000 :  3.26437604576
    Nearest to The Hitchhiker's Guide to the Galaxy (Hitchhiker's Guide to the Galaxy, #1): A Clockwork Orange, Catch-22, Middlesex,
    Nearest to Cinder (The Lunar Chronicles, #1): Seven Deadly Wonders (Jack West Jr, #1), The Fault in Our Stars, Embrace (The Violet Eden Chapters, #1),
    Nearest to The Kite Runner: Tuesdays with Morrie, A Clockwork Orange, Life of Pi,
    Nearest to The Glass Castle: The Bonesetter's Daughter, Peony in Love, The Other Boleyn Girl (The Plantagenet and Tudor Novels, #9),
    Nearest to Extremely Loud and Incredibly Close: The Poisonwood Bible, Lolita, The Omnivore's Dilemma: A Natural History of Four Meals,
    Nearest to Love in the Time of Cholera: The Catcher in the Rye, Pride and Prejudice, The Grapes of Wrath,
    Nearest to The Catcher in the Rye: Love in the Time of Cholera, Pride and Prejudice, Ancillary Sword (Imperial Radch, #2),
    Nearest to Crime and Punishment: Slaughterhouse-Five, Zen and the Art of Motorcycle Maintenance: An Inquiry Into Values, To Have and Have Not,
    Nearest to Animal Farm: The Good Earth (House of Earth, #1), Gone with the Wind, Infinite Jest,
    Nearest to Wicked: The Life and Times of the Wicked Witch of the West (The Wicked Years, #1): Ender's Game (Ender's Saga, #1), Atonement, The Secret Life of Bees,
    Nearest to The Perks of Being a Wallflower: To Kill a Mockingbird, Catch-22, The Curious Incident of the Dog in the Night-Time,
    Nearest to War and Peace: Harry Potter and the Deathly Hallows (Harry Potter, #7), Gravity's Rainbow, Nine Stories,
    Nearest to Brave New World: The Giver (The Giver, #1), Middlesex, Catch-22,
    Nearest to The Girl with the Dragon Tattoo (Millennium, #1): The Secret of the Unicorn (Tintin, #11), The Housekeeper and the Professor, Dewey: The Small-Town Library Cat Who Touched the World,
    Nearest to The Casual Vacancy: The Fault in Our Stars, The Prayer of Jabez:  Breaking Through to the Blessed Life, Barefoot Contessa Back to Basics,
    Nearest to Lord of the Flies: The Kite Runner, Snow, 1984,
    Average loss at step  82000 :  3.25000136065
    Average loss at step  84000 :  3.25605355531
    Average loss at step  86000 :  3.29702677715
    Average loss at step  88000 :  3.38729267067
    Average loss at step  90000 :  3.27407476926
    Nearest to The Hitchhiker's Guide to the Galaxy (Hitchhiker's Guide to the Galaxy, #1): A Clockwork Orange, Catch-22, Middlesex,
    Nearest to Cinder (The Lunar Chronicles, #1): Seven Deadly Wonders (Jack West Jr, #1), Embrace (The Violet Eden Chapters, #1), The Fault in Our Stars,
    Nearest to The Kite Runner: Tuesdays with Morrie, A Clockwork Orange, Lolita,
    Nearest to The Glass Castle: The Bonesetter's Daughter, Peony in Love, A Tree Grows in Brooklyn,
    Nearest to Extremely Loud and Incredibly Close: The Poisonwood Bible, Lolita, The Omnivore's Dilemma: A Natural History of Four Meals,
    Nearest to Love in the Time of Cholera: Pride and Prejudice, The Catcher in the Rye, War and Peace,
    Nearest to The Catcher in the Rye: Love in the Time of Cholera, Pride and Prejudice, The Divine Comedy,
    Nearest to Crime and Punishment: Slaughterhouse-Five, To Have and Have Not, Zen and the Art of Motorcycle Maintenance: An Inquiry Into Values,
    Nearest to Animal Farm: The Good Earth (House of Earth, #1), Gone with the Wind, Infinite Jest,
    Nearest to Wicked: The Life and Times of the Wicked Witch of the West (The Wicked Years, #1): Ender's Game (Ender's Saga, #1), Atonement, The Hitchhiker's Guide to the Galaxy (Hitchhiker's Guide to the Galaxy, #1),
    Nearest to The Perks of Being a Wallflower: Catch-22, To Kill a Mockingbird, The Curious Incident of the Dog in the Night-Time,
    Nearest to War and Peace: Harry Potter and the Deathly Hallows (Harry Potter, #7), Gravity's Rainbow, Love in the Time of Cholera,
    Nearest to Brave New World: Catch-22, The Giver (The Giver, #1), To Kill a Mockingbird,
    Nearest to The Girl with the Dragon Tattoo (Millennium, #1): The Secret of the Unicorn (Tintin, #11), The Housekeeper and the Professor, Dewey: The Small-Town Library Cat Who Touched the World,
    Nearest to The Casual Vacancy: Wonder, The Fault in Our Stars, The Prayer of Jabez:  Breaking Through to the Blessed Life,
    Nearest to Lord of the Flies: The Kite Runner, 1984, Pilgrim at Tinker Creek,
    Average loss at step  92000 :  3.24749968338
    Average loss at step  94000 :  3.21547119522
    Average loss at step  96000 :  3.20347243738
    Average loss at step  98000 :  3.19260094315
    Average loss at step  100000 :  3.22952575564
    Nearest to The Hitchhiker's Guide to the Galaxy (Hitchhiker's Guide to the Galaxy, #1): A Clockwork Orange, Catch-22, Middlesex,
    Nearest to Cinder (The Lunar Chronicles, #1): Seven Deadly Wonders (Jack West Jr, #1), The Fault in Our Stars, Embrace (The Violet Eden Chapters, #1),
    Nearest to The Kite Runner: Tuesdays with Morrie, A Clockwork Orange, 1984,
    Nearest to The Glass Castle: The Bonesetter's Daughter, Peony in Love, A Tree Grows in Brooklyn,
    Nearest to Extremely Loud and Incredibly Close: The Poisonwood Bible, Lolita, The Omnivore's Dilemma: A Natural History of Four Meals,
    Nearest to Love in the Time of Cholera: The Catcher in the Rye, Pride and Prejudice, The Great Gatsby,
    Nearest to The Catcher in the Rye: Love in the Time of Cholera, Pride and Prejudice, War and Peace,
    Nearest to Crime and Punishment: Slaughterhouse-Five, Zen and the Art of Motorcycle Maintenance: An Inquiry Into Values, To Have and Have Not,
    Nearest to Animal Farm: The Good Earth (House of Earth, #1), Gone with the Wind, Infinite Jest,
    Nearest to Wicked: The Life and Times of the Wicked Witch of the West (The Wicked Years, #1): Ender's Game (Ender's Saga, #1), The Hitchhiker's Guide to the Galaxy (Hitchhiker's Guide to the Galaxy, #1), Atonement,
    Nearest to The Perks of Being a Wallflower: Catch-22, To Kill a Mockingbird, The Curious Incident of the Dog in the Night-Time,
    Nearest to War and Peace: Harry Potter and the Deathly Hallows (Harry Potter, #7), Gravity's Rainbow, The Unbearable Lightness of Being,
    Nearest to Brave New World: The Giver (The Giver, #1), Catch-22, The Old Man and the Sea,
    Nearest to The Girl with the Dragon Tattoo (Millennium, #1): The Secret of the Unicorn (Tintin, #11), Dewey: The Small-Town Library Cat Who Touched the World, Love, Rosie,
    Nearest to The Casual Vacancy: Barefoot Contessa Back to Basics, The Fault in Our Stars, Wild: From Lost to Found on the Pacific Crest Trail,
    Nearest to Lord of the Flies: The Kite Runner, 1984, World War Z: An Oral History of the Zombie War,


## Step 4. Save the final embeddings for recommendation


```python
final_embeddings.shape
```




    (9951, 128)



Code for recommendation based on cosine similarity  >> https://howtoanalyse.github.io/recommender/Word2Vec-Gensim/

