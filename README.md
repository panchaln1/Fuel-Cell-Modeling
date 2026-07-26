# Fuel-Cell-Modeling
This is the python based model to simulate and optimize the membrane thickness of Electrolyte of Lithium-Aluminum Air/Hybrid Fuel Cell, Also it creates the voltage loss curves
A1: config/constants.py
 
"""
constants.py
 
Fundamental physical constants used in the
Hybrid Li-Al/O2 fuel cell model.
"""
 
class Constants:
    """
    Universal physical constants.
    """
 
    # Universal gas constant
    R = 8.314462618      # J mol^-1 K^-1
 
    # Faraday constant
    F = 96485.33212      # C mol^-1
 
 
CONST = Constants()
A2: config/parameters.py
 
"""
parameters.py
 
Electrochemical and material parameters
for the hybrid Li-Al/O2 fuel cell.
"""
 
import numpy as np
 
 
# ==========================================================
# Cell operating parameters
# ==========================================================
 
class CellParameters:
 
    """
    General fuel cell operating conditions.
    """
 
    # Operating current density
    # A/m2
    current_density = 1000.0
 
    # Limiting current density
    # A/m2
    limiting_current_density = 20000.0
 
 
CELL = CellParameters()
 
 
 
# ==========================================================
# Cathode parameters
# ==========================================================
 
class CathodeParameters:
 
    """
    Cathode kinetic parameters.
 
    Cathode material:
    La1-xSrxMnO3 (LSM)
    """
 
    # Number of electrons transferred
    n = 4
 
    # Charge transfer coefficient
    alpha = 0.5
 
    # Exchange current density
    # A/m2
    exchange_current_density = 1.0
 
 
CATHODE = CathodeParameters()
 
 
 
# ==========================================================
# Scandia Stabilized Zirconia electrolyte
# ==========================================================
 
class SCSZParameters:
 
    """
    Scandia stabilized zirconia electrolyte parameters.
 
    Electrolyte:
    (ZrO2)1-x(Sc2O3)x
    """
 
    # Electrolyte thickness
    # meter
    thickness = 5e-6
 
 
SCSZ = SCSZParameters()
 
 
 
# ==========================================================
# Temperature parameters
# ==========================================================
 
class TemperatureParameters:
 
    """
    Temperature range for analysis.
    """
 
    # Kelvin
    minimum = 700
 
    maximum = 1200
 
    points = 100
 
 
TEMP = TemperatureParameters()
A3: models/nernst.py
 
"""
nernst.py
 
Calculation of reversible cell voltage using
the Nernst equation.
"""
 
import numpy as np
 
from config.constants import CONST
 
 
class NernstModel:
 
    """
    Reversible cell voltage model.
    """
 
    def voltage(self, temperature):
 
        temperature = np.asarray(
            temperature,
            dtype=float
        )
 
        # Standard reversible voltage
        # Li-Al/O2 system reference value
 
        E0 = 2.32
 
        return E0 * np.ones_like(temperature)
A4: models/conductivity.py
 

"""
conductivity.py
 
Temperature-dependent ionic conductivity
model for solid electrolyte.
"""
 
import numpy as np
 
 
class ConductivityModel:
 
    """
    Arrhenius conductivity model.
    """
 
    def conductivity(self, temperature):
 
        temperature = np.asarray(
            temperature,
            dtype=float
        )
 
        # Pre-exponential conductivity
        sigma0 = 120.0       # S/m
 
        # Activation energy
        Ea = 85000.0         # J/mol
 
        R = 8.314462618
 
        conductivity = (
            sigma0 *
            np.exp(
                -Ea /
                (R * temperature)
            )
        )
 
        return conductivity
 

 

 

A5: models/activation.py
 
"""
activation.py
 
Cathode activation polarization model.
 
Tafel approximation:
ηact = RT/(αnF) ln(i/i0)
"""
 
import numpy as np
 
from config.constants import CONST
from config.parameters import CELL
from config.parameters import CATHODE
 
 
class ActivationModel:
 
    """
    Cathode activation polarization.
    """
 
    def activation_loss(
        self,
        temperature,
        current_density=None
    ):
 
        temperature = np.asarray(
            temperature,
            dtype=float
        )
 
        if current_density is None:
 
            current_density = (
                CELL.current_density
            )
 
        current_density = np.asarray(
            current_density,
            dtype=float
        )
 
        # Avoid logarithm singularity
 
        current_density = np.maximum(
            current_density,
            1e-12
        )
 
        eta = (
            CONST.R *
            temperature /
            (
                CATHODE.alpha *
                CATHODE.n *
                CONST.F
            )
        ) * np.log(
            current_density /
            CATHODE.exchange_current_density
        )
 
        return eta
A6: models/concentration.py
 
"""
concentration.py
 
Concentration polarization model
based on limiting current density.
"""
 
import numpy as np
 
from config.constants import CONST
from config.parameters import CELL
from config.parameters import CATHODE
 
 
class ConcentrationModel:
 
    """
    Mass-transfer polarization.
    """
 
    def concentration_loss(
        self,
        temperature,
        current_density=None
    ):
 
        temperature = np.asarray(
            temperature,
            dtype=float
        )
 
        if current_density is None:
 
            current_density = (
                CELL.current_density
            )
 
        current_density = np.asarray(
            current_density,
            dtype=float
        )
 
 
        ratio = (
            current_density /
            CELL.limiting_current_density
        )
 
 
        if np.any(ratio >= 1):
 
            raise ValueError(
                "Current density exceeds limiting current density."
            )
 
 
        eta = (
            -CONST.R *
            temperature /
            (
                CATHODE.n *
                CONST.F
            )
        ) * np.log(
            1-ratio
        )
 
 
        return eta
A7: models/electrolyte.py
 
"""
electrolyte.py
 
Electrolyte ohmic polarization model.
 
ηohmic = i*l/κ
"""
 
import numpy as np
 
from config.parameters import SCSZ
from config.parameters import CELL
 
 
class ElectrolyteModel:
 
    """
    Electrolyte resistance model.
    """
 
    def area_specific_resistance(
        self,
        conductivity
    ):
 
        conductivity = np.asarray(
            conductivity,
            dtype=float
        )
 
        return (
            SCSZ.thickness /
            conductivity
        )
 
 
    def ohmic_loss(
        self,
        conductivity,
        current_density=None
    ):
 
 
        if current_density is None:
 
            current_density = (
                CELL.current_density
            )
 
 
        current_density = np.asarray(
            current_density,
            dtype=float
        )
 
 
        ASR = (
            self.area_specific_resistance(
                conductivity
            )
        )
 
 
        return (
            current_density *
            ASR
        )
A8: models/cell_voltage.py
 
"""
cell_voltage.py
 
Complete hybrid Li-Al/O2 fuel cell voltage model.
 
Vcell = Erev - ηactivation - ηconcentration - ηohmic
"""
 
import numpy as np
 
from models.nernst import NernstModel
from models.activation import ActivationModel
from models.concentration import ConcentrationModel
from models.conductivity import ConductivityModel
from models.electrolyte import ElectrolyteModel
 
from config.parameters import CELL
 
 
class CellVoltageModel:
 
 
    def __init__(self):
 
        self.nernst = NernstModel()
 
        self.activation = ActivationModel()
 
        self.concentration = ConcentrationModel()
 
        self.conductivity = ConductivityModel()
 
        self.electrolyte = ElectrolyteModel()
 
 
 
    def voltage(
        self,
        temperature,
        current_density=None
    ):
 
        """
        Calculate cell voltage.
 
        Parameters
        ----------
        temperature :
            Operating temperature (K)
 
        current_density :
            Current density (A/m2)
 
        Returns
        -------
        Dictionary of voltage losses
        """
 
 
        if current_density is None:
 
            current_density = (
                CELL.current_density
            )
 
 
        current_density = np.asarray(
            current_density,
            dtype=float
        )
 
 
        conductivity = (
            self.conductivity.conductivity(
                temperature
            )
        )
 
 
        Erev = (
            self.nernst.voltage(
                temperature
            )
        )
 
 
        eta_activation = (
            self.activation.activation_loss(
                temperature,
                current_density
            )
        )
 
 
        eta_concentration = (
            self.concentration.concentration_loss(
                temperature,
                current_density
            )
        )
 
 
        eta_ohmic = (
            self.electrolyte.ohmic_loss(
                conductivity,
                current_density
            )
        )
 
 
        Vcell = (
            Erev
            -
            eta_activation
            -
            eta_concentration
            -
            eta_ohmic
        )
 
 
        return {
 
            "Current Density":
                current_density,
 
            "Erev":
                Erev,
 
            "Conductivity":
                conductivity,
 
            "Activation":
                eta_activation,
 
            "Concentration":
                eta_concentration,
 
            "Ohmic":
                eta_ohmic,
 
            "Cell Voltage":
                Vcell
        }
A9: analysis/polarization.py
 
"""
polarization.py
 
Generation of polarization curve and
power-density curve.
"""
 
import numpy as np
import pandas as pd
 
from models.cell_voltage import CellVoltageModel
from config.parameters import CELL
 
 
 
class PolarizationAnalysis:
 
 
    def __init__(self):
 
        self.model = CellVoltageModel()
 
 
 
    def generate(
        self,
        temperature,
        filename
    ):
 
 
        # Current density range
 
        current_density = np.linspace(
            0,
            CELL.limiting_current_density*0.95,
            100
        )
 
 
        voltage_results = []
 
 
        for current in current_density:
 
 
            result = self.model.voltage(
                temperature,
                current
            )
 
 
            voltage_results.append(
                result["Cell Voltage"]
            )
 
 
 
        voltage_results = np.array(
            voltage_results
        )
 
 
        power_density = (
            current_density *
            voltage_results
        )
 
 
 
        data = pd.DataFrame(
            {
 
            "Current Density (A/m²)":
                current_density,
 
            "Cell Voltage (V)":
                voltage_results,
 
            "Power Density (W/m²)":
                power_density
 
            }
        )
 
 
        data.to_csv(
            filename,
            index=False
        )
 
 
        return data
A10: Main Execution Script (run_model.py)
 
"""
run_model.py
 
Main program for hybrid Li-Al/O2
fuel cell simulation.
"""
 
 
from analysis.polarization import (
    PolarizationAnalysis
)
 
 
 
# Operating temperature
 
temperature = 678.15
# 405 °C
 
 
 
analysis = PolarizationAnalysis()
 
 
 
results = analysis.generate(
    temperature,
    "polarization_curve.csv"
)
 
 
 
print(
    "Polarization curve generated successfully."
)
 
 
print(results.head())
