# Code Dive: Liciotti et al. (2020) - Teaching a Machine to Recognize Human Activities

**Paper:** A Sequential Deep Learning Application for Recognising Human Activities in Smart Homes <br>
**Authors:** Liciotti, D., Bernardini, M., Romeo, L., & Frontoni, E. <br>
[**Official Repository**](https://github.com/danielelic/deep-casas) <br>
**Internal Wiki Analysis:** [Paper-Analysis-Liciotti-2020](https://github.com/ehkarabasbu/swe577-inclusive-participatory-smart-environments/wiki/Related-Work#liciotti-et-al-2020---a-sequential-deep-learning-application-for-recognising-human-activities-in-smart-homes)

---

## 1. Handling the Messy Sensor Data

First of all we won't be able to just feed raw sensor logs to a neural network. `data.py` here is their solution. It's the unglamorous but very important job of turning a chaotic stream of sensor events from the CASAS datasets into clean, ordered sequences that a model can actually learn from.

### `data.py` - `load_dataset` function

So how do they do it? The clever bit is using the "begin" and "end" markers in the logs to figure out which sensor events belong to a specific activity. This lets them gather all the little events and group them into one coherent story.

```python
# Found in: 08_cloned_repos/week3/liciotti_etal_2020/deep-casas/data.py

def load_dataset(filename):
    # ...
    # Find the start and end of an activity in the log
    for i, line in enumerate(database):
        # ...
        if 'begin' in des:
            activity = re.sub('begin', '', des)
            activities.append(activity)
        if 'end' in des:
            # ...
            activity = ''
    # ...
    # Now, group all the sensor events (XX) that happened
    # during that activity (y) into a single sequence (x)
    for i, y in enumerate(YY):
        if i > 0:
            if y == YY[i - 1]:
                x.append(XX[i]) # Keep adding to the sequence
            else:
                Y.append(y)
                X.append(x)     # The activity changed, so finish the last sequence
                x = [XX[i]]     # And start a new one
    # ...
    return X, Y, dictActivities
```

This whole data prep stage is fundamental. For our research, it's a clear guide for how to wrangle continuous, real-world sensor data into something a machine can understand. If our system is going to get user behavior, we'll need a process just like this to divide the raw data stream into activities that actually mean something.

---

## 2. The Model's Brain: An LSTM Network

This is where the learning happens. `models.py` uses Keras to build the LSTM network that figures out the patterns in the sensor data sequences.

### `models.py` - `get_LSTM` function

The model itself is surprisingly straightforward. But there's one argument in the `Embedding` layer that's super important for us: `mask_zero=True`. What does it do? It tells the model to just ignore all the zero-padding we have to add to make the sequences the same length. This is a big deal, because in real life, activities don't all take the same amount of time.

```python
# Found in: 08_cloned_repos/week3/liciotti_etal_2020/deep-casas/models.py

from keras.layers import Dense, LSTM
from keras.layers.embeddings import Embedding
from keras.models import Sequential

def get_LSTM(input_dim, output_dim, max_lenght, no_activities):
    model = Sequential(name='LSTM')
    # The mask_zero=True here is the key.
    model.add(Embedding(input_dim, output_dim, input_length=max_lenght, mask_zero=True))
    model.add(LSTM(output_dim))
    model.add(Dense(no_activities, activation='softmax'))
    return model
```

This model gives us the "what" (what's the user doing?) that was missing from Dave's "where" (the BIM system). That little `mask_zero` argument is a key technical idea for our own framework—it makes the model tough enough to handle messy, variable human behavior. We can pretty much lift this architecture directly to add activity recognition to our own smart environment.

---

## 3. Putting It All Together: Training and Testing

We can think of `train.py` here as the conductor of an orchestra. It loads the data, builds the model, starts the training, and checks the results. 

### `train.py` - The Main Loop

The script uses a `StratifiedKFold` cross-validation strategy. It's a reliable way to test a model. It makes sure the results you're seeing are real and not just a lucky coincidence from one particular train-test split.

```python
# Found in: 08_cloned_repos/week3/liciotti_etal_2020/deep-casas/train.py

if __name__ == '__main__':
    # ...
    # Loop through the different CASAS datasets
    for dataset in data.datasetsNames:
        X, Y, dictActivities = data.getData(dataset)

        # Use k-fold to split the data for robust testing
        kfold = StratifiedKFold(n_splits=3, shuffle=True, random_state=seed)
        
        for train, test in kfold.split(X, Y):
            # ... (build and compile the model)

            # Train it
            model.fit(X_train_input, Y[train], validation_split=0.2, epochs=epochs, ...)

            # Lnd see how well it did
            scores = model.evaluate(X_test_input, Y[test], ...)
            
            # ... (print out the results)
```

The way this script is structured is a great blueprint for our own experiments. Using cross-validation is a lesson we need to take to heart. It gives you a much more honest measure of how well your model is *actually* doing. This script is what makes their research something we can build on.