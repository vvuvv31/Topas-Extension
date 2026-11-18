# Hadron LET weighting

`myHadronLET` scores charged hadrons and ions with atomic number Z >= 1.
For `WeightBy = "dose"`, numerator and denominator now use the same
step energy deposit (`Edep`) as the weight:

```
LETd = sum(Edep * electronic_stopping_power / density) / sum(Edep)
```

The electronic stopping power is evaluated at the mean of the pre-step
and post-step kinetic energies. The default `Order = 1` is assumed above.
`WeightBy = "fluence"` or `"track"` continues to use step length.

Previously, the denominator added the kinetic energy of newly created
secondary electrons, while the numerator used Edep alone. The resulting
ratio used inconsistent weights. This correction changes dose-weighted
LET values; do not combine corrected results with the previous definition.

Empty bins still produce zero LET, but no longer count as a missing
denominator warning unless the numerator is nonzero. String conversion
uses `G4StrUtil::to_lower` for current Geant4 compatibility.

## Regression check

Build TOPAS with `myHadronLET.cc`, `myHadronLET.hh`,
`myHadronLET_Denominator.cc`, and `myHadronLET_Denominator.hh` in the
extension directory. Run the supplied carbon-ion EM-only check:

```sh
topas examples/HadronLET_EdepRegression.txt
python3 - <<'PY'
import math
import struct
from pathlib import Path

den = struct.unpack('d', Path('denominator.bin').read_bytes())[0]
edep = struct.unpack('d', Path('carbon_edep.bin').read_bytes())[0]
assert den > 0 and math.isfinite(den)
assert math.isclose(den, edep, rel_tol=1e-10), (den, edep)
print('Edep weighting regression passed:', den, edep)
PY
```

The EM-only physics keeps the charged-ion species fixed, so a separate
EnergyDeposit scorer filtered to carbon-12 provides an independent check
of the denominator. This checks the weighting correction, not clinical
accuracy or all stopping-power assumptions.
