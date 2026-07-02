# Autism GAN Pipeline: High-Level Design (HLD)

Based on the excellent explanation you shared, here are the two visual diagrams that map perfectly to the described architecture. I have explicitly drawn the split exactly where we fixed it, so you can see how the test set is completely isolated from both the GAN and the classifier training.

## HLD Part 1: Data-Generation Flow

This diagram shows how raw fMRI data is processed into functional connectivity vectors, how the data is strictly split to prevent leakage, and how the GAN learns the statistical shape of the training subset to generate new matrices.

```mermaid
graph TD
    subgraph "1. Feature Extraction"
        A[Preprocessed fMRI Scans] -->|Pearson Correlation| B["200x200 FC Matrices"]
        B -->|Flatten upper triangle| C["19,900-dim Real Vectors"]
    end
    
    subgraph "2. Strict 80/20 Split (The Leakage Fix)"
        C -->|80% split| D["Real Training Vectors"]
        C -->|20% split| E["Real Test Vectors (Vault)"]
    end

    subgraph "3. WGAN-GP Training & Synthesis"
        F["Random Noise (128-dim) + Label"] --> G["Generator Network"]
        D -->|Real Samples| H["Discriminator Network"]
        G -->|Fake Samples| H
        H -.->|Loss Signal| G
        
        G -->|After 2000 Epochs| I["Synthetic 19,900-dim Vectors"]
        I -->|Reconstruct matrix| J["Synthetic 200x200 FC Matrices"]
    end
    
    style E fill:#f9d0c4,stroke:#333,stroke-width:2px
    style D fill:#d4edda,stroke:#333,stroke-width:2px
    style J fill:#cce5ff,stroke:#333,stroke-width:2px
```

## HLD Part 2: Classification Flow

This diagram shows how the synthetic vectors are used strictly as a supplement to the training data. The "Vaulted" test set is brought out at the very end to evaluate both the baseline and augmented models fairly.

```mermaid
graph TD
    subgraph "Training Data Sources"
        A["80% Real Train Vectors"]
        B["Synthetic Vectors (from GAN)"]
    end
    
    subgraph "Model Training"
        A -->|Baseline| C["Train Baseline Models (SVM, RF, etc.)"]
        A --> D["Augmented Training Pool"]
        B --> D
        D -->|Augmented| E["Train Augmented Models"]
    end
    
    subgraph "Evaluation (Strictly Unseen Data)"
        F["20% Real Test Vectors (Vault)"]
        F --> G["Evaluate Baseline Models"]
        F --> H["Evaluate Augmented Models"]
        
        C -.-> G
        E -.-> H
    end
    
    G --> I{"Compare AUC & Accuracy"}
    H --> I
    
    style F fill:#f9d0c4,stroke:#333,stroke-width:2px
    style A fill:#d4edda,stroke:#333,stroke-width:2px
    style B fill:#cce5ff,stroke:#333,stroke-width:2px
    style I fill:#fff3cd,stroke:#333,stroke-width:2px
```

### The "Asterisk" Addressed
As noted in the text you shared, the red "Vault" box (the 20% Real Test Vectors) is mathematically invisible to the Generator and Discriminator in HLD Part 1. Because we modified the code to enforce this split *before* Step 2 runs, the GAN never gets to peek at the test subjects, making the final evaluation comparison (the yellow box in Part 2) 100% trustworthy.
