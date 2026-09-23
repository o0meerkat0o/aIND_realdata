# aIND on real channel flow (CoMSAIL, informal)

Applies aIND (Arranz & Lozano-Durán, JFM 2024; code: https://github.com/Computational-Turbulence-Group/aIND) to the UPC Re_tau = 180 channel flow DNS (figshare DOI 10.25452/figshare.plus.30636068).

Open `aIND_real_data.ipynb` in Colab. Expects the raw data in `MyDrive/comsail/Mesh/` and `MyDrive/comsail/0-150_LETOTs/`. The small extract (`aIND_extract.npz`) is included here so the notebook can run without the raw files.

## Status

**Working**
- Repo runs after a small PyTorch compatibility patch (`ReduceLROnPlateau`'s `verbose` kwarg was removed in newer versions).
- Synthetic validation (paper §2.4) reproduces the published result: corr(Φ_I, f) = 0.989, E_I = 0.975 vs. 0.990 exact.
- Real data loads correctly: axis order (z, y, x) confirmed from mesh coordinate ranges, ghost cells trimmed, and wall shear stress τ_w computed from ν = 1/180 comes out with mean ≈ 1.02 (expected ≈ 1.0). Mean profile, near-wall streaks, and τ_w pattern all look physical.
- Full pipeline runs end to end on the real data: train → decompose → MI checks.

**Not working**
- At a lag of 1 snapshot, aIND fails in one of two ways: Φ_I collapses to near zero (E_I ≈ 0.001), or with a lighter MI weight the MSE loss sits flat around 1.0 for the whole run and never actually fits.

## What I think the issue is

aIND probably isn't broken — it may be answering a question that has no signal in it as currently set up.

- **The time lag is likely too large.** The dataset README gives 0.5 large-eddy turnover times between saved snapshots, which works out to roughly 90 wall units per step in the best case (if the files are consecutive saves). The paper trains at a lag of about 25 wall units. The actual spacing between our files hasn't been confirmed, so it could be worse.
- **Source and target are paired at the same (x, z) point.** Over 90+ wall units, near-wall turbulent structures convect a few hundred wall units downstream. So the wall stress at a point one snapshot later has little to do with the velocity at that same point now — the structure that "caused" it has already moved elsewhere.
- **Both failure modes are consistent with this one cause.** Everything is standardized before training, so an MSE stuck at ≈ 1.0 means "the target explains essentially none of the source." If there's genuinely no relationship at that lag, Φ_I → 0 is the *correct* output, not a bug.

## What's untested

Whether the lag/pairing hypothesis is actually right. The notebook has two diagnostic cells for this:
1. Correlation between Φ and Ψ as a function of lag, with and without a streamwise shift (to account for convection).
2. A lag-0 control fit with the MI term turned off, to confirm the network can fit a relationship when one exists.

If lag-0 correlation is high and lag-1 drops to near zero, that confirms the lag is the problem, not the training. That's the next thing to check before tuning the model further.

## Next steps if the hypothesis holds

1. Confirm the actual physical time spacing between snapshot files (check file attributes, or the dataset README).
2. Either pull finer-spaced snapshots (~25 wall units apart) or add a streamwise offset between source and target to account for convection.
3. Only then revisit tuning the MI weight / learning rate.
4. Secondary check: source-plane y+ vs. what the paper used.
