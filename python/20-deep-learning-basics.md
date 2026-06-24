# 🎯 Topic 20: Deep Learning Basics

> **Data Science Interview — Neural Networks & Deep Learning Fundamentals**  
> From perceptrons to transformers — the essential deep learning knowledge for DS/DA roles at big tech. Covers architectures, training mechanics, regularization, and the critical "when to use DL vs. traditional ML" decision framework.

---

## Table of Contents

1. [Neural Network Fundamentals](#neural-network-fundamentals)
2. [Activation Functions](#activation-functions)
3. [Forward Propagation](#forward-propagation)
4. [Backpropagation](#backpropagation)
5. [Loss Functions](#loss-functions)
6. [Optimizers](#optimizers)
7. [Regularization Techniques](#regularization-techniques)
8. [Convolutional Neural Networks (CNNs)](#convolutional-neural-networks-cnns)
9. [Recurrent Neural Networks & LSTMs](#recurrent-neural-networks--lstms)
10. [Transfer Learning](#transfer-learning)
11. [Common Architectures](#common-architectures)
12. [Deep Learning vs. Traditional ML](#deep-learning-vs-traditional-ml)
13. [Interview Talking Points & Scripts](#interview-talking-points--scripts)
14. [Common Interview Mistakes](#common-interview-mistakes)
15. [Rapid-Fire Q&A](#rapid-fire-qa)
16. [Summary Cheat Sheet](#summary-cheat-sheet)

---

## Neural Network Fundamentals

A neural network is a **universal function approximator** — given enough neurons, it can learn any continuous mapping from inputs to outputs.

```python
import numpy as np

def perceptron(inputs, weights, bias):
    """Single neuron: weighted sum + activation"""
    z = np.dot(inputs, weights) + bias  # linear combination
    return 1 if z > 0 else 0            # step activation
```

**Building Blocks**: Weights (scale inputs), Bias (shifts activation threshold), Layers (input → hidden → output).

> **Critical Insight:** A single hidden layer can theoretically approximate any function (Universal Approximation Theorem), but **depth** is exponentially more parameter-efficient than width for learning hierarchical features.

| Term | Meaning | Example |
|------|---------|---------|
| Dense/Fully Connected | Every neuron connects to all in previous layer | `Dense(64)` |
| Parameters | Learnable values (weights + biases) | 64×32 + 32 = 2,080 |
| Hyperparameters | Set before training, not learned | Learning rate, batch size |
| Epoch | One full pass through training data | 50 epochs |

---

## Activation Functions

Activation functions introduce **non-linearity** — without them, stacking layers equals a single linear transformation.

| Function | Formula | Range | Pros | Cons | Use Case |
|----------|---------|-------|------|------|----------|
| **ReLU** | max(0, x) | [0, ∞) | Fast, avoids vanishing gradient | "Dying ReLU" (stuck at 0) | Default for hidden layers |
| **Leaky ReLU** | max(0.01x, x) | (-∞, ∞) | No dying neurons | Extra hyperparameter | When ReLU neurons die |
| **Sigmoid** | 1/(1+e^(-x)) | (0, 1) | Probabilistic output | Vanishing gradient | Binary output layer |
| **Tanh** | (e^x-e^(-x))/(e^x+e^(-x)) | (-1, 1) | Zero-centered | Vanishing gradient | RNNs (legacy) |
| **Softmax** | e^(xi)/Σe^(xj) | (0,1), sums to 1 | Probability distribution | Expensive | Multi-class output |

> **Critical Insight:** ReLU is the default for hidden layers because its gradient is 1 for positive inputs — no multiplicative shrinkage through layers, unlike sigmoid (max gradient 0.25).

```python
import tensorflow as tf

model = tf.keras.Sequential([
    tf.keras.layers.Dense(128, activation='relu'),      # hidden
    tf.keras.layers.Dense(64, activation='relu'),       # hidden
    tf.keras.layers.Dense(10, activation='softmax')     # multi-class output
])
```

---

## Forward Propagation

Forward propagation is the **computation flow** from input to output: linear transform then activation, repeated per layer.

For layer l: `z[l] = W[l] · a[l-1] + b[l]` then `a[l] = activation(z[l])`

```python
def forward_pass(X, parameters):
    """Simple 2-layer forward propagation"""
    W1, b1, W2, b2 = parameters['W1'], parameters['b1'], parameters['W2'], parameters['b2']
    z1 = np.dot(W1, X) + b1
    a1 = np.maximum(0, z1)              # ReLU
    z2 = np.dot(W2, a1) + b2
    a2 = 1 / (1 + np.exp(-z2))         # Sigmoid output
    return a2, {'z1': z1, 'a1': a1, 'z2': z2, 'a2': a2}
```

> **Critical Insight:** Forward propagation is deterministic — same weights + input = same output. The "learning" happens only during backpropagation.

---

## Backpropagation

Backpropagation is the **chain rule applied recursively** — computing the gradient of the loss with respect to each weight by propagating errors backward.

**Intuition**: (1) Compute loss → (2) Ask each weight "how much did you contribute?" → (3) Update proportionally.

```python
def backward_pass(X, Y, cache, parameters):
    """Backpropagation for 2-layer network"""
    m = X.shape[1]
    a1, a2 = cache['a1'], cache['a2']
    W2 = parameters['W2']
    
    dz2 = a2 - Y                                    # output gradient
    dW2 = (1/m) * np.dot(dz2, a1.T)
    db2 = (1/m) * np.sum(dz2, axis=1, keepdims=True)
    
    dz1 = np.dot(W2.T, dz2) * (cache['z1'] > 0)   # chain rule + ReLU derivative
    dW1 = (1/m) * np.dot(dz1, X.T)
    db1 = (1/m) * np.sum(dz1, axis=1, keepdims=True)
    return {'dW1': dW1, 'db1': db1, 'dW2': dW2, 'db2': db2}
```

> **Critical Insight:** The vanishing gradient problem occurs when gradients shrink exponentially through many layers (sigmoid/tanh). Solutions: ReLU activations, skip connections (ResNet), and proper initialization.

---

## Loss Functions

| Loss Function | Use Case | Key Property |
|---------------|----------|--------------|
| **MSE** | Regression | Penalizes large errors heavily (squared) |
| **MAE** | Regression (outlier-robust) | Constant gradient, robust to outliers |
| **Binary Cross-Entropy** | Binary classification | Heavily penalizes confident wrong answers |
| **Categorical Cross-Entropy** | Multi-class classification | Works with softmax output |
| **Huber Loss** | Regression (balanced) | MSE when small error, MAE when large |

```python
# Keras loss function selection
model.compile(optimizer='adam', loss='binary_crossentropy')       # binary
model.compile(optimizer='adam', loss='categorical_crossentropy')  # multi-class
model.compile(optimizer='adam', loss='mse')                       # regression
```

> **Critical Insight:** Cross-entropy produces large gradients when confidently wrong (predict 0.99, truth is 0). MSE would produce near-zero gradients in that scenario due to sigmoid saturation — never use MSE for classification.

---

## Optimizers

| Optimizer | Key Idea | Pros | Cons | When to Use |
|-----------|----------|------|------|-------------|
| **SGD** | W -= lr × gradient | Good generalization | Slow, oscillates | Robust final convergence |
| **Momentum** | Adds velocity term | Accelerates consistent directions | Extra hyperparameter | When SGD is too slow |
| **RMSprop** | Adapts lr per parameter | Handles sparse gradients | Can be unstable | RNNs, non-stationary |
| **Adam** | Momentum + RMSprop | Fast, adaptive | May generalize worse | Default starting choice |
| **AdamW** | Adam + decoupled weight decay | Better generalization | Slightly complex | Transformers, fine-tuning |

```python
import torch.optim as optim

# Adam — default starting point
optimizer = optim.Adam(model.parameters(), lr=0.001)

# SGD with momentum — often better final performance
optimizer = optim.SGD(model.parameters(), lr=0.01, momentum=0.9, weight_decay=1e-4)

# Learning rate scheduling
scheduler = optim.lr_scheduler.ReduceLROnPlateau(optimizer, patience=5, factor=0.5)
```

> **Critical Insight:** Adam converges faster but SGD+momentum often achieves better generalization. Start with Adam for prototyping, switch to SGD+momentum for production when you can tune the learning rate schedule.

---

## Regularization Techniques

| Technique | How It Works | Typical Values |
|-----------|--------------|----------------|
| **Dropout** | Randomly zeroes neurons during training | Rate: 0.2-0.5 |
| **Batch Normalization** | Normalizes layer inputs to zero mean, unit var | After linear, before activation |
| **L2 (Weight Decay)** | Adds λΣw^2 to loss | 1e-4 to 1e-2 |
| **L1 (Lasso)** | Adds λΣ\|w\| to loss (sparse weights) | 1e-5 to 1e-3 |
| **Early Stopping** | Stop when val loss increases | Patience: 5-10 epochs |
| **Data Augmentation** | Create modified training samples | Domain-specific |

```python
model = tf.keras.Sequential([
    tf.keras.layers.Dense(256, activation='relu',
                          kernel_regularizer=tf.keras.regularizers.l2(0.001)),
    tf.keras.layers.BatchNormalization(),
    tf.keras.layers.Dropout(0.3),
    tf.keras.layers.Dense(128, activation='relu'),
    tf.keras.layers.Dropout(0.3),
    tf.keras.layers.Dense(10, activation='softmax')
])

early_stop = tf.keras.callbacks.EarlyStopping(
    monitor='val_loss', patience=10, restore_best_weights=True)
model.fit(X_train, y_train, validation_split=0.2, epochs=100, callbacks=[early_stop])
```

> **Critical Insight:** Dropout is an implicit ensemble of 2^n sub-networks. BatchNorm both regularizes AND accelerates training by reducing internal covariate shift — one of the most impactful DL innovations.

---

## Convolutional Neural Networks (CNNs)

CNNs exploit **spatial structure** through parameter sharing and translation invariance.

| Operation | Purpose | Key Parameters |
|-----------|---------|----------------|
| **Convolution** | Extract local features via learned filters | Kernel size, stride, padding |
| **Pooling** | Downsample, add translation invariance | Pool size (typically 2x2) |
| **Feature Maps** | Each filter detects one pattern type | Number of filters |

```python
import torch.nn as nn

class SimpleCNN(nn.Module):
    def __init__(self, num_classes=10):
        super().__init__()
        self.features = nn.Sequential(
            nn.Conv2d(3, 32, kernel_size=3, padding=1),  # RGB input
            nn.ReLU(), nn.MaxPool2d(2, 2),
            nn.Conv2d(32, 64, kernel_size=3, padding=1),
            nn.ReLU(), nn.MaxPool2d(2, 2),
            nn.Conv2d(64, 128, kernel_size=3, padding=1),
            nn.ReLU(), nn.AdaptiveAvgPool2d((1, 1))
        )
        self.classifier = nn.Linear(128, num_classes)
    
    def forward(self, x):
        x = self.features(x)
        return self.classifier(x.view(x.size(0), -1))
```

> **Critical Insight:** CNNs learn hierarchical features automatically — early layers detect edges, middle layers detect textures/parts, deep layers detect objects. Weight sharing means the same filter slides everywhere, encoding translation invariance with far fewer parameters.

---

## Recurrent Neural Networks & LSTMs

RNNs process **sequential data** by maintaining hidden state across time steps. Vanilla RNNs suffer from vanishing gradients — LSTMs solve this with **gates**.

| Gate | Purpose | Intuition |
|------|---------|-----------|
| **Forget Gate** | Discard from cell state | "Should I forget this?" |
| **Input Gate** | Store new information | "Is this worth remembering?" |
| **Output Gate** | What to output | "What's relevant right now?" |

```python
class SentimentLSTM(nn.Module):
    def __init__(self, vocab_size, embed_dim=128, hidden_dim=256):
        super().__init__()
        self.embedding = nn.Embedding(vocab_size, embed_dim)
        self.lstm = nn.LSTM(embed_dim, hidden_dim, num_layers=2,
                           batch_first=True, dropout=0.3, bidirectional=True)
        self.fc = nn.Linear(hidden_dim * 2, 1)  # *2 for bidirectional
    
    def forward(self, x):
        embedded = self.embedding(x)
        _, (hidden, _) = self.lstm(embedded)
        hidden = torch.cat((hidden[-2], hidden[-1]), dim=1)
        return torch.sigmoid(self.fc(hidden))
```

> **Critical Insight:** Transformers have largely replaced LSTMs — self-attention attends to ANY position directly without the sequential bottleneck. LSTMs remain useful for streaming/online scenarios processing one token at a time.

---

## Transfer Learning

Use **pretrained models** as starting points — dramatically reduces data and compute needs.

| Scenario | Strategy | Example |
|----------|----------|---------|
| Small data, similar domain | Freeze base, train new head | Medical images + ImageNet model |
| Small data, different domain | Fine-tune last few layers | Custom text + BERT last layers |
| Large data, similar domain | Fine-tune entire network (low lr) | More product images + ResNet |
| Large data, different domain | Train from scratch | Satellite images, millions of samples |

```python
base_model = tf.keras.applications.ResNet50(
    weights='imagenet', include_top=False, input_shape=(224, 224, 3))
base_model.trainable = False  # freeze

model = tf.keras.Sequential([
    base_model,
    tf.keras.layers.GlobalAveragePooling2D(),
    tf.keras.layers.Dense(256, activation='relu'),
    tf.keras.layers.Dropout(0.3),
    tf.keras.layers.Dense(5, activation='softmax')
])
model.compile(optimizer=tf.keras.optimizers.Adam(1e-3), loss='categorical_crossentropy')
model.fit(train_data, epochs=10)

# Fine-tune: unfreeze last 20 layers, lower lr
base_model.trainable = True
for layer in base_model.layers[:-20]:
    layer.trainable = False
model.compile(optimizer=tf.keras.optimizers.Adam(1e-5), loss='categorical_crossentropy')
```

> **Critical Insight:** Transfer learning is the single most impactful practical DL technique. State-of-the-art results on small datasets (hundreds of examples) become achievable. Always start here.

---

## Common Architectures

| Architecture | Key Innovation | Domain |
|-------------|----------------|--------|
| **ResNet** (2015) | Skip connections (residual learning) | Vision |
| **Transformer** (2017) | Self-attention mechanism | NLP → everything |
| **BERT** (2018) | Bidirectional pre-training + fine-tuning | NLP understanding |
| **GPT** (2018-2024) | Autoregressive pre-training at scale | Text generation |
| **ViT** (2020) | Image patches as tokens in Transformer | Vision |

### Transformer Self-Attention: `Attention(Q, K, V) = softmax(QK^T / sqrt(d_k)) V`

- **Q** (Query): "What am I looking for?" | **K** (Key): "What do I contain?" | **V** (Value): "What info do I give?"

```python
import torch.nn.functional as F

class SelfAttention(nn.Module):
    def __init__(self, embed_dim, num_heads=8):
        super().__init__()
        self.num_heads = num_heads
        self.head_dim = embed_dim // num_heads
        self.qkv = nn.Linear(embed_dim, 3 * embed_dim)
        self.proj = nn.Linear(embed_dim, embed_dim)
    
    def forward(self, x):
        B, T, C = x.shape
        qkv = self.qkv(x).reshape(B, T, 3, self.num_heads, self.head_dim).permute(2,0,3,1,4)
        q, k, v = qkv[0], qkv[1], qkv[2]
        attn = F.softmax((q @ k.transpose(-2,-1)) / (self.head_dim**0.5), dim=-1)
        return self.proj((attn @ v).transpose(1,2).reshape(B, T, C))
```

> **Critical Insight:** Skip connections let gradients flow directly through the network without degradation — that's why ResNet-152 trains better than a plain 20-layer net. It's about gradient flow, not capacity.

---

## Deep Learning vs. Traditional ML

| Factor | Favor Deep Learning | Favor Traditional ML |
|--------|--------------------|--------------------|
| **Data volume** | 100K+ samples | < 10K samples |
| **Data type** | Images, text, audio, video | Tabular/structured |
| **Feature engineering** | DL learns representations | Hand-crafted features work |
| **Interpretability** | Not required | Critical (regulatory) |
| **Compute** | GPU available | Limited resources |
| **Latency** | Batch OK | Sub-ms required |

**When NOT to use DL**: (1) Tabular data < 50K rows — XGBoost wins. (2) Interpretability required. (3) No GPU. (4) Small data without pretrained model. (5) Simple linear relationships.

> **Critical Insight:** For tabular data, gradient-boosted trees outperform DL in the vast majority of benchmarks. DL shines on unstructured data where it learns representations no human could hand-engineer.

---

## Interview Talking Points & Scripts

### "Explain backpropagation in simple terms"

> *"Backpropagation is how neural networks learn from mistakes. After making a prediction, we calculate the error. Then we work backward through the network using the chain rule from calculus, asking each weight: 'How much did you contribute to this error?' Each weight gets a gradient telling it which direction to adjust. We update weights proportionally to reduce the error. It's an efficient algorithm to compute the derivative of the loss with respect to every weight simultaneously — without it, training deep networks would be computationally infeasible."*

### "When would you NOT use deep learning?"

> *"I'd avoid deep learning in several scenarios. For tabular data, XGBoost consistently wins — especially under 50K rows. When interpretability matters for regulators or stakeholders, simpler models are far more transparent. With limited data and no pretrained model, DL overfits quickly. Under compute constraints or sub-millisecond latency requirements, simpler models win. My rule: start with the simplest model that could work, escalate to deep learning only when data type or complexity demands it."*

### "How would you approach a new DL project?"

> *"I establish a strong baseline first — often logistic regression or a pretrained model with minimal fine-tuning. Then I incrementally add complexity: use a standard architecture (ResNet for vision, BERT for text), apply transfer learning, and validate each change improves validation metrics. I set up experiment tracking, use early stopping, and reserve a held-out test set evaluated only once. The biggest mistake is jumping to a complex architecture before establishing whether DL is even the right tool."*

### "Explain the bias-variance tradeoff in deep learning"

> *"In deep learning, large networks can have near-zero training error while still generalizing — contradicting classical statistics. This is explained by implicit regularization from SGD and loss landscape structure. If training loss is high, the model underfits — add capacity. If training loss is low but validation loss is high, it overfits — add regularization (dropout, weight decay, data augmentation) or get more data."*

---

## Common Interview Mistakes

| # | Mistake ❌ | Correction ✅ |
|---|-----------|--------------|
| 1 | "Deep learning always beats ML" | DL excels on unstructured data; GBTs dominate tabular |
| 2 | Choosing architecture before understanding data | Start with data exploration, establish baselines first |
| 3 | "More layers = better model" | Deeper nets are harder to train without skip connections |
| 4 | Using sigmoid in hidden layers | Use ReLU as default for hidden layers |
| 5 | Forgetting to normalize inputs | Standardize inputs; use BatchNorm between layers |
| 6 | "Adam is always the best optimizer" | Adam converges fast; SGD often generalizes better |
| 7 | Training without a validation set | Always split data; early stop on validation loss |
| 8 | Ignoring learning rate tuning | LR is the most impactful hyperparameter — tune first |
| 9 | "Dropout helps during inference" | Dropout is ONLY active during training |
| 10 | Not considering transfer learning | Always check for pretrained models in your domain |

---

## Rapid-Fire Q&A

**Q1: Why does ReLU help with vanishing gradients?**  
A: Gradient is 1 for positive inputs (vs. sigmoid's max 0.25) — no exponential shrinkage through layers.

**Q2: Parameter vs. hyperparameter?**  
A: Parameters are learned (weights). Hyperparameters are set beforehand (learning rate, architecture).

**Q3: Why is BatchNorm so effective?**  
A: Reduces internal covariate shift, allows higher learning rates, and acts as a regularizer.

**Q4: MSE vs. cross-entropy for classification?**  
A: Always cross-entropy — MSE produces near-zero gradients when sigmoid saturates.

**Q5: What makes Adam different from SGD?**  
A: Adam maintains adaptive per-parameter learning rates using momentum and variance of gradients.

**Q6: How does dropout prevent overfitting?**  
A: Forces redundancy — implicit ensemble of 2^n sub-networks. No single neuron can memorize.

**Q7: Why are skip connections important?**  
A: Create shortcut gradient paths, preventing degradation and enabling 100+ layer training.

**Q8: What is softmax temperature?**  
A: Scales logits before softmax: high T = uniform/random, low T = peaked/confident. Used in distillation.

**Q9: Why is transfer learning effective?**  
A: Lower layers learn universal features (edges, grammar) that transfer — only top layers need new data.

**Q10: Key insight of Transformers?**  
A: Self-attention lets every position attend to every other in O(1) sequential steps — no RNN bottleneck.

**Q11: How to diagnose underfitting vs. overfitting?**  
A: Underfitting: high train + val loss. Overfitting: low train loss, high val loss (gap between them).

**Q12: Why do CNNs use weight sharing?**  
A: Same filter slides everywhere — encodes translation invariance, reduces params dramatically.

---

## Summary Cheat Sheet

```
+-----------------------------------------------------------------------+
|                    DEEP LEARNING BASICS CHEAT SHEET                    |
+-----------------------------------------------------------------------+
|                                                                       |
|  NETWORK = Layers of: z = Wx + b  -->  a = activation(z)            |
|                                                                       |
|  ACTIVATIONS:                                                         |
|    Hidden layers --> ReLU    |  Binary output --> Sigmoid             |
|    Multi-class output --> Softmax                                     |
|                                                                       |
|  TRAINING: Forward pass --> Loss --> Backward pass --> Update W       |
|                                                                       |
|  LOSS: Regression --> MSE/Huber  |  Classification --> Cross-Entropy |
|                                                                       |
|  OPTIMIZER: Start Adam(lr=1e-3), tune lr first                       |
|                                                                       |
|  REGULARIZATION: BatchNorm + Dropout(0.3) + EarlyStopping + L2       |
|                                                                       |
|  ARCHITECTURES:                                                       |
|    Images --> CNN (ResNet via transfer learning)                      |
|    Text --> Transformer (BERT/GPT via fine-tuning)                    |
|    Tabular --> DON'T use DL (use XGBoost/LightGBM)                   |
|                                                                       |
|  TRANSFER LEARNING: Always try pretrained model FIRST                |
|                                                                       |
|  KEY NUMBERS:                                                         |
|    Batch: 32-128  |  LR: 1e-4 to 1e-2  |  Dropout: 0.2-0.5         |
|                                                                       |
+-----------------------------------------------------------------------+
```

---

*Part of the [Data Science Interview Prep](../README.md) series — Topic 20 of 25*
