# python_mini-project_IMPA-with-lipophilicity

## Does chemical lipophilicity help improve predictions for cell morphological responses?
<br><br>
### Purpose
Based on the tutorial “Use IMPA for unseen drug prediction BBBC021” by Palma et al. (2024), this project aimed to:
1.	Reproduce a model that predicts the cell phenotypic response images in Cell Painting assays after chemical treatment.
2.	Test whether the model’s predictive performance improves when an additional input variable, log*D*<sub>lipw</sub> (pH=7.4), is included.
<br><br>
### Motivation
The cell membrane is a common target site for chemicals. Among environmental chemicals, some with high lipophilicity or low target specificity can induce toxicity through baseline toxicity, caused by direct interactions with the membrane. This led me to question: **If certain phenotypic changes arise from non-specific interactions driven by chemical lipophilicity rather than from chemical-specific actions, could they be explained by lipophilicity of individual chemicals?**
Thus, I wanted to observe whether lipophilicity could induce common patterns of phenotypic changes after chemical exposure, and whether a lipophilicity metric could improve predictive power of the existing model. Instead of the commonly used logKow, I used log*D*<sub>lipw</sub> (pH=7.4), which considers pH and is more physiologically relevant to the real cell membrane environment.
<br><br>
### Methods
Based on Palma et al.’s tutorial “Use IMPA for unseen drug prediction BBBC021”, I tried to maximize reuse of the uploaded checkpoint, with the following updates:

1. Image Resizing

  The uploaded “BBBC021 all” dataset only had images at 96 × 96 resolution. The pretrained checkpoint was trained on 128 × 128 images, so directly using 96 × 96 inputs caused size mismatch errors. To resolve this, I created a wrapper at the dataloader stage to resize the images, while keeping the dataset and checkpoint themselves unchanged.


2. Additional Input: log*D*<sub>lipw</sub> (pH=7.4)

The log*D*<sub>lipw</sub> (pH=7.4) values for the chemicals were obtained as described in Lee et al. (2021) and recorded in “Physicochemical properties_details.xlsx”. Figure 1 shows an excerpt of pKa, logKlipw, and final log*D*<sub>lipw</sub> (pH=7.4) values for the chemicals used in this project.

<img width="940" height="127" alt="image" src="https://github.com/user-attachments/assets/4f9b08ca-f801-46d0-99c9-16c8f6a3de8a" />

**Figure 1. Excerpt from the table of physicochemical properties of 35 chemicals.**

To maintain checkpoint compatibility and test only the additive effect of log*D*<sub>lipw</sub> (pH=7.4), I applied an **additive adapter layer approach**: log*D*<sub>lipw</sub> (pH=7.4) was added residually to the representation rather than concatenated as an independent variable. This could limit representational power but allows direct comparison. If log*D*<sub>lipw</sub> (pH=7.4) contributes meaningfully, concatenation could be tested later. log*D*<sub>lipw</sub> (pH=7.4) values were z-score normalized before being applied to the model.

3. Model Optimization & Comparisons

The original checkpoint included 88 domains, but the uploaded dataset had only 35 compounds. After excluding OOD compounds and control (DMSO), 24 compounds were available for training. So, I applied **transfer learning** to optimize the IMPA model on these 24 compounds.

4. Experiments:  
(1) Experiment 1 (LP-baseline): Freeze backbone (generator, style encoder, mapping network) and train only head (discriminator) to adapt the model to operate with 24 domains. Serves as baseline through linear probing (LP), keeping the representation fixed while varying only the head.  
(2) Experiment 2 (Full FT): Fine-tune (FT) both backbone and head to test improvement under reduced domain conditions.  
(3) Experiment 3 (Full FT + log*D*<sub>lipw</sub>): Fine-tune backbone and head with log*D*<sub>lipw</sub> (pH=7.4) included to test effect of additional feature.  

| Experiment |	Input	| Training scope	| Comparison purpose |
| :----: | :----: | :----: | :----: |
|1) LP-baseline|	fingerprint|	Head only|	Transfer baseline|
|2) Full FT|	fingerprint|	backbone+head|	Effect of fine-tuning|
|3) Full FT + log*D*<sub>lipw</sub>| fingerprint+ log*D*<sub>lipw</sub>|	backbone+head|	Effect of additional feature|

(LP: Linear probing; FT: Fine-tuning, log*D*<sub>lipw</sub> when pH = 7.4)
<br><br>
### Performance Comparison

 <img width="741" height="454" alt="image" src="https://github.com/user-attachments/assets/e3eef00a-0b1f-4c6c-bd64-c48a1e01ee22" />

**Figure 2. Performance measures of trained models (mean ± std, n = 3, * = p < 0.05)**

In Figure 2, both Experiment 2 and 3 with FT showed improved Fréchet Inception Distance (FID) compared to baseline (Experiment 1). Experiment 3 had the lowest FID, but the improvement over Experiment 2 was not statistically significant. Recall scores were similar across all experiments, with no significant differences.  
•	Conclusion: **Fine-tuning improved performance but adding log*D*<sub>lipw</sub> (pH=7.4) as input did not significantly enhance predictive power.**  
•	Note: FID was used as in the original IMPA, but due to small sample size in this project, Kernel Inception Distance (KID) may have been more appropriate.  
<br><br>
### Limitations & Future Improvements
1.	Limited number of compounds used for training (35 total; 24 for learning).
2.	Lipophilicity may explain common phenotypic patterns arising from non-specific interactions with membranes, but it may be insufficient to account for the diverse phenotypic changes induced by compound-specific actions at the organelle level. Baseline toxicity effects may primarily impact outer membranes, but with our still limited understanding, it is difficult to extend this explanation to organelle membranes.
3.	The compounds used in this project had well-defined mode of actions (MOAs) with high target specificity, thus they likely act on specific targets rather than membranes. In contrast, many environmental chemicals have weak and/or multiple MOAs, or act mainly on membranes. For example, comparing log*D*<sub>lipw</sub> (pH=7.4) distributions of test compounds from Lee et al. (2021, n = 392 environmental chemicals) and this project (n = 34, excluding DMSO) shows that this dataset covers hydrophilic ranges well but not lipophilic ranges (−1 < log*D*<sub>lipw</sub> (pH=7.4) < 5.2, see Figure 3).

 <img width="756" height="433" alt="image" src="https://github.com/user-attachments/assets/9b6bc733-acc2-48ee-96ce-10890328dbc4" />

**Figure 3. Distribution of log*D*<sub>lipw</sub> (pH=7.4) from Lee et al. (2021, n = 392) and this project (n = 34, excluding DMSO used as a control)**

  Therefore, training on a wider variety of compounds could improve the predictive value of log*D*<sub>lipw</sub> (pH=7.4) for certain groups, especially for highly lipophilic compounds that primarily act on membranes. Still, log*D*<sub>lipw</sub> (pH=7.4) may be too limited to cover all MOAs and resulting phenotypic responses.

4.	Using 3D-based fingerprints (instead of 2D) may better capture diverse molecular features and improve predictions.
5.	Alternative models to IMPA (style transfer and GAN-based) could be explored—e.g., flow matching models (direction of the ESPOD project).
<br><br>
________________________________________
*I used ChatGPT (OpenAI, 2025) to understand limitations of the IMPA model and how flow matching-based generative models can address them.

Limitations of the IMPA Model
1.	Prone to mode collapse and unstable training.  
2.	May fail to cover the full data distribution, focusing only on partial patterns.  
3.	Generated image quality not sufficiently close to real cell images (FID-based).  
4.	No explicit denoising process, limiting fine cellular detail.  
5.	Style-transfer approach restricts ability to represent continuous perturbation effects (interpolation).  

Complementary Features of Flow Matching-based Generative Models
1.	Stable training – learns probability flows directly via ordinary differential equations (ODEs); no discriminator, no mode collapse.
2.	Higher sample quality – smoother mapping from noise to real distribution; reported ~35% FID improvement over IMPA.
3.	Continuous perturbation simulation – naturally generates interpolations/extrapolations between perturbations.
4.	Better separation of batch effect and perturbation effect – absorbs artifacts into noise flow, isolating biological effects.
5.	Sampling efficiency – requires fewer steps than diffusion models, allowing faster generation.

<br><br>
### References
Lee J, Braun G, Henneberger L, König M, Schlichting R, Scholz S, Escher BI. 2021. Critical Membrane Concentration and Mass-Balance Model to Identify Baseline Cytotoxicity of Hydrophobic and Ionizable Organic Chemicals in Mammalian Cell Lines. Chem Res Toxicol 34:2100-2109. DOI: 10.1021/acs.chemrestox.1c00182.

OpenAI. 2025. GPT-5, ChatGPT model. OpenAI, San Francisco, CA. Available at: https://chat.openai.com/ (accessed in August-September 2025).

Palma A, Theis FJ, Lotfollahi M. 2025. Predicting cell morphological responses to perturbations using generative modeling. Nat Commun 16:505. DOI: 10.1038/s41467-024-55707-8.

Palma A, Theis FJ, Lotfollahi M. 2025. IMPA: Predicting cell morphological responses to perturbations (GitHub repository). GitHub. Available at: https://github.com/theislab/impa (accessed in August-September, 2025)

