# HiSA Solver

The solver is a C++ based tool for computing compressible transonic and supersonic flow. HiSA was developed to provide users with an efficient and robust aerodynamic toolset that can be extended to model a wide range of physics. ... [read more](https://hisa.gitlab.io/)

- [**Installion (locally)**](https://hisa.gitlab.io/download.html)
- [**ocumentation**](https://hisa.gitlab.io/doc.html)  

## Singularity (recommended)

To make singularity container follow [this](https://github.com/darshan315/OpenFOAM_HiSA_PyTorch). The singularity image contains installation of pre-compiled version of OpenFOAM by ESI, and installation and compilation of HiSA solver.  

## Test Case

To execute the test case with HiSA solver refer the "rhoCF_set1_alpha4_sa_ref1" case implemented in `./test_cases/HiSA/` directory. To execute the simulation with HiSA solver, following modifications are required compared to OpenFOAM simulation settings.  

## Solver Setup

### Initial and Boundary conditions  

<details>
<summary markdown="spawn"> Click to expand! </summary>
Along with the OpenFOAM boundary conditions, HiSA enables aditional boundary condition options, which is mentioned in chaper 2.1 in [HiSA-Documentation](https://hisa.gitlab.io/doc.html).

For the given simulation the HiSA specific "Characteristic-based boundary conditions" are also used. Here, characteristic boundary condition requires the freestream conditions of all the primary variables. It is therefore suggested that a file `0.org/include/freestreamConditions` should be created which contains the freestream velocity, pressure and temperature. This field can also be included at the beginning of each of the primary variable field files as follows,

```c++
#include    "include/freestreamConditions"
```

The `freestreamConditions` file for 4° AoA is defined as follows,

```c++
U            (254.59829691 17.80324723 0);
p             1e5;
T             293;
#inputMode    merge
```

#### 0.org/U

HiSA specific "`characteristicFarfieldVelocity`" boundary conditions for inlet and outlet.

```c++
"(inlet|outlet)"
    {
        type            characteristicFarfieldVelocity;
        #include        "include/freestreamConditions"
        value           $internalField;
    }
```

#### 0.org/p

HiSA specific "`characteristicFarfieldPressure`" boundary conditions for inlet and outlet.

```c++
    "(inlet|outlet)"
    {
        type            characteristicFarfieldPressure;
        #include        "include/freestreamConditions"
        value           $internalField;
    }
```

#### 0.org/T

HiSA specific "`characteristicFarfieldTemperature`" boundary conditions for inlet and outlet.

```c++
    "(inlet|outlet)"
    {
    type            characteristicFarfieldTemperature;
        #include        "include/freestreamConditions"
        value           $internalField;
    }
```

HiSA specific "characteristicWallTemperature" boundary conditions for wall(airfoil).

```c++
    airfoil
    {
        type            characteristicWallTemperature;
        value           $internalField;
    }
```

</details>

### Numerical Schemes (fvScheme)

<details>
<summary markdown="spawn"> Click to expand! </summary>

Shock-capturing scheme is selected under `fluxScheme` as follows,

```c++
fluxScheme           AUSMPlusUp;
lowMachAusm          false;
```

Temporal discretistion is specified under `ddtSchemes` as follows,

```c++
ddtSchemes
{
    default          bounded dualTime rPseudoDeltaT steadyState;
}
```

Gradient schemes are specified under `gradSchemes` as follows,

```c++
gradSchemes
{
    default          faceLeastSquares linear;
    grad(nuTilda)    cellLimited Gauss linear 0.9;
}
```

Divergence schemes are specified under `divSchemes` as follows,  
  
```c++  
divSchemes
{
    default          none;
    div(tauMC)       Gauss linear;
    div(phi,nuTilda) bounded Gauss upwind phi;
}
```

Laplacian schemes are specified under `laplacianSchemes` as follows,

```c++
laplacianSchemes
{
    default                     Gauss linear limited corrected 0.33;
    laplacian(muEff,U)          Gauss linear compact;
    laplacian(alphaEff,e)       Gauss linear compact;
}
```

Interpolation Schemes are specified in the fvSchemes file under the `interpolationSchemes` as follows,

```c++
interpolationSchemes
{
    default          linear;
    reconstruct(rho) wVanLeer;
    reconstruct(U)   wVanLeer;
    reconstruct(T)   wVanLeer;
}
```

</details>

### Numerical Solvers (fvSolution)

<details>
<summary markdown="spawn"> Click to expand! </summary>

The turbulence equations need to be specified under `solvers` as,

```c++
solvers
{
    "(k|nuTilda)"
    {
        solver          smoothSolver;
        smoother        symGaussSeidel;
        tolerance       1e-10;
        relTol          0.1;
        minIter         1;
    }

    yPsi
    {
        solver          GAMG;
        smoother        GaussSeidel;
        cacheAgglomeration true;
        nCellsInCoarsestLevel 10;
        agglomerator    faceAreaPair;
        mergeLevels     1;
        tolerance       1e-8;
        relTol          0;
    }
}
```

The degree of relaxation of the turbulence equations are introduced when considering viscous analysis. A value of 0.5 is recommended,

```c++
relaxationFactors
{
    equations
    {
        nuTilda         0.5;
        k               0.5;
        omega           0.5;
    }
}

```

Currently for HiSA, the only option for the coupled solver is GMRES, which selects the generalised
minimal residual method of [Saad and Schultz](https://epubs.siam.org/doi/10.1137/0907058). The settings required by the GMRES solver are contained in the GMRES sub-subdictionary of the flowSolver subdictionary and are described as,

```c++
flowSolver
{
    solver            GMRES;
    GMRES
    {
        inviscidJacobian  LaxFriedrichs;
        viscousJacobian   laplacian;
        preconditioner    LUSGS;

        maxIter           20;
        nKrylov           8;
        solverTolRel      1e-1 (1e-1 1e-1 1e-1) 1e-1;
    }
}
```

With the iterative implicit method a dual-time stepping approach is employed and the following
settings are contained in the pseudoTime dictionary to control the iterations in pseudo-time (also
known as ‘outer correctors’),

```c++
pseudoTime
{

    pseudoTol          1e-8 (1e-8 1e-8 1e-8) 1e-8;
    pseudoCoNum        1.0;
    pseudoCoNumMax     1.0;
    localTimestepping  true;
}
```

</details>
