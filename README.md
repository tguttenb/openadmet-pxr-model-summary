# Activity Prediction — Modeling Summary

**Challenge:** OpenADMET PXR Induction Challenge\
**Primary endpoint:** pEC50 dose-response prediction

This project developed a diverse molecular modeling ensemble for predicting PXR activation potency from chemical structure. The modeling strategy combined 2D graph neural networks, 3D molecular foundation models, conformer-based geometric representations, learned embeddings, classical cheminformatics descriptors, and tabular meta-learning layers.

The central goal was not to rely on a single molecular representation, but to build a broad set of complementary models that captured orthogonal chemical signals: graph topology, learned molecular fingerprints, 3D conformer geometry, nearest-neighbor structure-activity relationships, scaffold-aware generalization behavior, and systematic model residual patterns.

## Data and Validation Strategy

The primary training target was the pEC50 dose-response endpoint. Model development used out-of-fold validation rather than simple random train/test splits, because scaffold leakage can substantially overstate performance in molecular property prediction.

A custom scaffold-aware splitter was implemented to create validation folds that better reflected the challenge setting. Compounds were grouped by molecular scaffold so that highly related analogs were not freely distributed across train and validation folds. The splitter was designed to preserve scaffold separation while also maintaining approximate balance across the activity distribution, reducing the risk that any single fold was dominated by unusually active, inactive, or chemically narrow compounds.

This scaffold-aware cross-validation framework was used throughout model development to generate clean out-of-fold predictions, train stacking layers, assess model diversity, and identify systematic residual behavior.

## Core Molecular Model Families

Multiple molecular learning architectures were trained on the pEC50 data to capture complementary structure-activity signals.

### Chemprop Models

Chemprop directed message-passing neural networks were trained directly on the pEC50 endpoint. These models provided a strong graph-based representation of molecular topology and local chemical environments. Chemprop embeddings were extracted and reused downstream as learned molecular features.

Chemprop served two roles:

1.  As a direct supervised predictor of pEC50 activity.

2.  As an embedding generator whose internal learned representations could be fused with other tabular and geometric features.

### DeepChem AttentionFP Models

DeepChem AttentionFP models were also trained on the pEC50 training data. AttentionFP provided an attention-based graph neural representation, allowing the model to learn atom and bond importance patterns associated with PXR activity.

AttentionFP embeddings were extracted and used as downstream features in stacked learning layers. These embeddings added diversity relative to Chemprop because the architecture learns molecular information through a different graph aggregation and attention mechanism.

### Uni-Mol Models

Uni-Mol models were trained to incorporate 3D molecular structure and conformer geometry. These models provided a pretrained 3D molecular transformer representation, intended to capture spatial features that are not fully represented by 2D fingerprints or graph topology alone.

Uni-Mol features were especially valuable as a complementary modeling channel because PXR binding and activation may depend on molecular shape, flexibility, and 3D pharmacophoric arrangement.

### TorchMD Models

TorchMD-based models were implemented using conformer-derived molecular geometry. These models added another 3D learning pathway focused on molecular coordinates and geometric relationships.

The TorchMD models contributed a separate class of learned conformer-aware features, helping the ensemble capture shape-sensitive and geometry-sensitive activity patterns that may not be visible to purely 2D models.

### Conformer-Weighted 3D Prediction Aggregation

For the 3D conformer-based models, predictions were generated across a conformer ensemble rather than relying on a single molecular geometry. For each molecule, up to 30 conformers were examined, allowing the model to sample alternative low-energy spatial arrangements that the compound could plausibly adopt. The individual conformer-level predictions were then aggregated using a probability-weighted averaging strategy, where conformers judged to be more physically plausible contributed more strongly to the final molecular prediction. In practice, this meant using conformer likelihood or relative-energy-derived weighting rather than a simple unweighted mean, so the final 3D prediction reflected both the activity signal predicted from each geometry and the estimated probability that the molecule would occupy that geometry. This helped preserve useful conformer diversity while reducing the influence of unlikely or high-energy conformations.

## Tabular Fused Head Features

In addition to learned neural embeddings, a large fused tabular feature layer was developed. These features were used as final-layer model inputs and as auxiliary features alongside neural embeddings.

The fused feature set included:

> Tainimoto nearest-neighbor similarity features;
>
> scaffold and local chemical neighborhood information;
>
> conformer-derived geometric descriptors;
>
> RDKit molecular descriptors;
>
> ECFP6 fingerprint PCA features;
>
> ECFP4 fingerprint PCA features;
>
> Mol2Vec molecular embeddings;
>
> learned embeddings from Chemprop and AttentionFP models;
>
> learned 3D representations from Uni-Mol and TorchMD models.

This fused-head approach allowed the final models to combine mechanistically different information sources: local analog similarity, global molecular descriptors, fingerprint-based structural variation, learned GNN representations, and 3D geometry.

The final learning layers used these combined features to exploit both high-capacity learned representations and lower-variance tabular structure-activity signals.

## Single-Concentration Embedding Transfer

A second major component of the modeling strategy was to reuse information from the single-concentration activity datasets.

Separate modeling pipelines were built for the single-concentration datasets at the **8e-6** level and the **3e-5** level. These were treated as related but distinct activity regimes rather than collapsed into a single auxiliary endpoint.

For each concentration level, dedicated Chemprop and DeepChem AttentionFP models were trained. The purpose was not only to predict the single-concentration response directly, but also to learn molecular embeddings specialized to that assay condition.

The learned embeddings from these single-concentration models were then inferred back onto the main pEC50 training compounds. These transferred embeddings became additional features for the primary pEC50 prediction task.

This created an auxiliary representation learning workflow:

1.  Train Chemprop and AttentionFP models on each single-concentration dataset.

2.  Extract learned molecular embeddings from those models.

3.  Apply the trained embedding models to the main pEC50 training and test compounds.

4.  Use the transferred embeddings as features in the final pEC50 model stack.

5.  Train an XGBoost final learning layer on the combined pEC50 feature matrix.

This allowed the final pEC50 models to benefit from related assay information without directly treating the single-concentration values as identical to dose-response potency.

## XGBoost Final Learning Layer

The final stacked models used XGBoost as a high-capacity tabular learning layer over the combined representation set.

Inputs to the XGBoost layer included:

> out-of-fold predictions from base models;
>
> Chemprop pEC50 embeddings;
>
> AttentionFP pEC50 embeddings;
>
> Uni-Mol representations;
>
> TorchMD-derived features;
>
> transferred Chemprop and AttentionFP embeddings from the single-concentration datasets;
>
> RDKit descriptors;
>
> ECFP PCA features;
>
> Mol2Vec features;
>
> nearest-neighbor features;
>
> conformer geometry features.

XGBoost was used because the final feature matrix contained a heterogeneous mixture of dense neural embeddings, sparse/fingerprint-derived structure signals, descriptor features, and model prediction features. Tree boosting is well-suited to this type of mixed tabular representation and can learn nonlinear interactions between model families.

## Ensemble and OOF Feature Development

A broad ensemble of out-of-fold model predictions was generated across the different model families. These OOF predictions were used for:

> estimating model generalization under scaffold-aware validation;
>
> measuring model complementarity;
>
> training final stacked learners;
>
> identifying compounds where the ensemble was systematically biased;
>
> building diagnostic classifiers for residual direction.

The ensemble was intentionally diverse. Chemprop and AttentionFP captured graph-based molecular learning, Uni-Mol and TorchMD captured 3D and conformer-aware signal, and the tabular features captured nearest-neighbor, descriptor, fingerprint, and molecular embedding information.

## Chronic Overprediction / Underprediction Classifier

After generating the ensemble of OOF predictions, a residual-direction analysis was performed. The goal was to identify compounds where the models were not merely noisy, but consistently biased in the same direction.

A “chronic over/underprediction” label was created using OOF training instances where the ensemble models uniformly agreed in their residual direction - meaning the models consistently overpredicted or consistently underpredicted the observed pEC50 value.

A separate classifier was then trained to predict this systematic residual behavior using molecular features independent of the final prediction value. The classifier used:

> RDKit descriptor features;
>
> conformer-derived geometric features;
>
> molecular structure and shape-related tabular features.

This classifier was then applied to the test set. Its logit scores were used as a post-processing signal to identify test compounds likely to be chronically overpredicted or underpredicted by the main ensemble.

Rather than making large manual corrections, the classifier output was used conservatively. Final predictions received only small directional adjustments, with the maximum absolute adjustment capped at **0.20 pEC50 units**.

This correction layer was designed to address systematic ensemble bias while limiting the risk of overfitting or destabilizing otherwise strong predictions.

## What the Modeling Strategy Tried to Capture

The overall modeling framework attempted to capture several distinct sources of predictive signal:

### 2D chemical topology

Chemprop and AttentionFP modeled atom-bond graph structure and local molecular environments.

### 3D molecular geometry

Uni-Mol and TorchMD incorporated conformer-based spatial information, molecular shape, and geometric arrangement.

### Local SAR / analog behavior

Nearest-neighbor features captured whether a test compound resembled active or inactive regions of the training set.

### Classical cheminformatics signal

RDKit descriptors, ECFP4 PCA, ECFP6 PCA, and Mol2Vec provided lower-variance tabular structure information.

### Auxiliary assay transfer

Single-concentration Chemprop and AttentionFP embeddings transferred related activity information into the primary pEC50 task.

### Systematic residual behavior

The over/underprediction classifier modeled when the full ensemble was likely to be directionally biased.

## Final Modeling Approach

The final approach was a stacked, scaffold-aware ensemble built from:

> Chemprop pEC50 models;
>
> DeepChem AttentionFP pEC50 models;
>
> Uni-Mol 3D molecular models;
>
> TorchMD conformer-based models;
>
> transferred embeddings from single-concentration Chemprop models;
>
> transferred embeddings from single-concentration AttentionFP models;
>
> RDKit molecular descriptors;
>
> ECFP4 and ECFP6 PCA features;
>
> Mol2Vec embeddings;
>
> nearest-neighbor features;
>
> conformer geometry features;
>
> XGBoost final learning layers;
>
> conservative residual-direction post-processing.

The core design principle was representation diversity. Instead of assuming that any single architecture could fully model PXR activity, the pipeline combined graph learning, 3D molecular modeling, auxiliary assay embedding transfer, cheminformatics descriptors, nearest-neighbor SAR features, and systematic residual correction.

The resulting system was intended to generalize better to novel scaffolds by combining multiple partially independent views of molecular activity and by explicitly modeling cases where the ensemble was likely to be directionally biased.

### Final Ensemble Distribution Re-Scaling

After the grand ensemble established a stable rank ordering of compounds, the final predictions were re-scaled to address distributional compression. Because the ensemble combined many partially correlated model families, the averaged predictions became overly conservative, with the final output distribution narrower than the observed pEC50 distribution in the training data. To correct this, the final ensemble predictions were blended toward a distribution calibrated from the original training labels, increasing the spread while preserving the rank structure learned by the ensemble. This post-processing step was intended to recover realistic dynamic range after rank order had already been established, so highly ranked compounds were allowed to move higher and low-ranked compounds lower without making large molecule-specific manual adjustments.

## Summary

This project implemented a broad molecular prediction stack for PXR pEC50 modeling. The approach moved beyond a single GNN or fingerprint model and instead used a multi-representation ensemble incorporating Chemprop, DeepChem AttentionFP, Uni-Mol, TorchMD, RDKit descriptors, ECFP PCA features, Mol2Vec, nearest-neighbor features, and conformer geometry.

A custom scaffold-aware splitter was used to produce more realistic validation folds and reliable out-of-fold features. Related single-concentration datasets were used as auxiliary representation-learning sources, with embeddings transferred back into the main pEC50 task. Final predictions were generated through XGBoost stacking layers, followed by a conservative over/underprediction correction based on classifier logits capped at 0.20 pEC50 units.

The final modeling strategy was therefore not a single model, but an integrated molecular representation and stacking framework designed to capture 2D structure, 3D geometry, auxiliary assay signal, local SAR, and systematic residual behavior.
