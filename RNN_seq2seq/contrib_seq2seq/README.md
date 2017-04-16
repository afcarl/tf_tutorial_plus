# API guide for [tf.contrib.seq2seq](https://github.com/tensorflow/tensorflow/blob/r1.1/tensorflow/contrib/seq2seq/)

## 1. Classes

### 1) Helper

#### `__init__(inputs, sequence_length, time_major=False)`
- `input_tas = tf.TensorArray.unstack(inputs)`

#### `initialize()`
```
finished = [False, False, ...] (batch_size)
if all(finished):
  next_inputs = zero-padded with same shape
else:
  next_inputs = input_tas[0]
return (finished, next_inputs)
```

#### `sample(time, outputs)`
```
sample_ids = tf.argmax(outputs, axis=-1, tf.int32)
```

#### `next_inputs(time, outputs, state)`
```
next_time = time + 1
finished = (next_time > sequence_length) (check if each batch is completed)
if all(finished):
  next_inputs = zero-padded with same shape
else:
  next_inputs = input_tas[next_time]
return (finished, next_inputs, state)
```

### 2) DecoderOutput
`namedtuple("DecoderOutput", ("rnn_output", "sample_id")))`

### 3) Decoder

#### `__init__(cell, helper, initial_state, output_layer=None)`
- output_layer: an instance of [`tf.layers.Layer`](https://github.com/tensorflow/tensorflow/blob/master/tensorflow/python/layers/core.py)
- ex) `output_layer = tf.layers.core.Dense(output_layer_depth, use_bias=False)`

#### `_rnn_output_size()`
```
if output_layer is None:
  return size
else:
  return output_layer._compute_output_shape[1:]
```

#### `output_size`
```
return DecoderOutput(_rnn_output_size(), TensorShape([]))
```

#### `initialize()`
```
return decoder._helper.initialize() + (decoder._initial_state, )
```

#### step(time, inputs, state)
```
cell_outputs, cell_state = cell(inputs, state)
if output_layer is not None:
        cell_outputs = output_layer(cell_outputs)
sample_ids = helper.sample(time, cell_outputs) # sometimes cell_state is needed
(finished, next_inputs, next_state) = helper.next_inputs(time, cell_outputs, cell_state, sample_ids)
outputs = DecoderOutput(cell_outputs, sample_ids)
return (outputs, next_state, next_inputs, finished)
```

## 2. Functions

### 1) dynamic_decode

#### Args:
- decoder: a [Decoder](###3\)-decoder) instance
- output_time_major=False
- impute_finished=False
- maximum_iterations=32
- swap_memory=False

#### Pseudocode
```
finished, inputs, state = decoder.initialize()

time = 0

outputs_ta = TensorArray(size=decoder.output_size)

while not all(finished):
  outputs, state, inputs, finished = decoder.step(time, inputs, state)

  if maximum_iterations is not None and time + 1 >= maximum_iterations:
      finished = True

  # if finished!
  # => zero out all remaining outputs
  if impute_finished:
    outputs = zero_outputs

  outputs_ta[time] = outputs

  time += 1

final_outputs = outputs_ta.stack()
final_state = state

# time_major => batch_major
# [max_dec_len x batch_size x hidden_size] => [batch_size x max_dec_len x hidden_size]
if not output_time_major:
  final_outputs = tf.transpose(final_outputs, [1,0,2])

return final_outputs, final_state

```