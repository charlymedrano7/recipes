.. highlight:: none

****************************
First calculation with DFTB+
****************************

[Input: `recipes/basics/firstcalc/`]

This chapter should serve as a tutorial guiding you through your first
calculation with DFTB+. As an example, the equilibrium geometry of the water
molecule will be calculated. As with all simulation tools, the process consists
of three steps:

* telling DFTB+ what to do,
* running DFTB+,
* analysing the results.


Providing the input
===================

DFTB+ accepts the input in the Human-readable Structured Data (HSD) format. The
input file must be called `dftb_in.hsd`.  The input file used in this example
looks as follows::

  Geometry = GenFormat {
  3 C
    O H

    1 1  0.00000000000E+00 -0.10000000000E+01  0.00000000000E+00
    2 2  0.00000000000E+00  0.00000000000E+00  0.78306400000E+00
    3 2  0.00000000000E+00  0.00000000000E+00 -0.78306400000E+00
  }

  Driver = GeometryOptimization {
    Optimizer = Rational {}
    MovedAtoms = 1:-1
    MaxSteps = 100
    OutputPrefix = "geom.out"
    Convergence {GradAMax = 1E-4}
  }

  Hamiltonian = DFTB {
    Scc = Yes
    SlaterKosterFiles {
      O-O = "../../slakos/mio-ext/O-O.skf"
      O-H = "../../slakos/mio-ext/O-H.skf"
      H-O = "../../slakos/mio-ext/H-O.skf"
      H-H = "../../slakos/mio-ext/H-H.skf"
    }
    MaxAngularMomentum {
      O = "p"
      H = "s"
    }
  }

  Options {}

  Analysis {
    CalculateForces = Yes
  }

  ParserOptions {
    ParserVersion = 12
  }

The order of the specified blocks in the HSD input is arbitrary. You are free to
capitalise the keywords and any physical units as you like, since they are
case-insensitive. This is not valid however for string values, especially if
they are specifying file names.

.. _gen_format:

Geometry
--------

The ``Geometry`` block contains types and coordinates of the atoms in your
system.  The geometry of the system in the sample input file is provided in the
so called "gen" format, which was the traditional geometry input format of the
DFTB method. The formal description of this format can be found in the DFTB+
manual.  In the current example, the geometry is ::

  Geometry = GenFormat {
    3  C                   # 3 atoms, non-periodic cluster
     O H                   # Two elements, 1 - O, 2 - H
    #  Index Type  Coordinates
         1    1    0.00000000000E+00  -0.10000000000E+01   0.00000000000E+00
         2    2    0.00000000000E+00   0.00000000000E+00   0.78306400000E+00
         3    2    0.00000000000E+00   0.00000000000E+00  -0.78306400000E+00
  }

which specifies a cluster of 3 atoms of chemical types O and H (cluster
geometries having open boundary conditions as distinct from periodic supercell
geometries). The Cartesian coordinates of the atoms in the "gen" format are
given in Angstroms.  The first column of integers contains the sequential
numbering of the atoms in the system (the actual values are ignored by the
parser).  The second column contains the type of each atom, given as the
position of the appropriate element in the element list of the second line of
the "gen" data.  The ``GenFormat{}`` is not the only way to specify the
geometry, you should check the manual for other methods.

As demonstrated above, it is possible to put arbitrary comments in the HSD input
after a hash-mark (``#``) character. Everything between this character and the
end of the current line is ignored by the parser.

Very often, the geometry is stored in an external file. To save you the copying
and pasting from that file into the `dftb_in.hsd` file, you can use the file
inclusion feature of the HSD format::

  Geometry = GenFormat {
    <<< "geometry.gen"
  }

The ``<<<`` operator includes the specified file as raw text data. (The file is
not checked for any HSD constructs.) In the example above, the file
`geometry.gen` *must* be in gen format.


Driver
------

After having specified the geometry of your system, you should decide what DFTB+
will do with that geometry. The ``Driver`` environment determines how the
geometry should be changed (if at all) during the calculation. If you only would
like to make a static calculation, you must either set it to an empty value
like ::

  Driver {}   # Empty value for the driver

or omit the ``Driver`` block completely from `dftb_in.hsd`.

In the current example ::

  # Perform rational function based optimisation
  Driver = GeometryOptimization {
    Optimizer = Rational {}
    MovedAtoms = 1:-1               # Move all atoms in the system
    MaxSteps = 100                  # Stop after maximal 100 steps
    OutputPrefix = "geom.out"       # Final geometry in geom.out.{xyz,gen}
    Convergence {GradAMax = 1E-4}   # Stop if maximal force below 1E-4 H/a0
  }

the molecule is relaxed using a rational function based optimiser. The
entire range of atoms from the first (atom 1) until and including the
last (-1) is allowed to move. Instead of ``1:-1`` you could also have
written::

  MovedAtoms = 1:3               # Atoms from the 1st until the 3rd

or ::

  MovedAtoms = O H               # Select O and H atoms.

or ::

  MovedAtoms = 1 2 3              # Explicitely listing all atom numbers.


In our case the geometry optimisation continues as long as the maximum component
of the force acting on the moving atoms is bigger than 1e-4 atomic units
(Hartree per Bohr radius). Numeric values are by default interpreted to be in
atomic units. However the HSD format offers the possibility of using alternative
units by specifying a unit modifier before the equals sign. This is given in
square brackets. For example instead of the original atomic units, you could
have used ::

  GradAMax [eV/AA] = 5.14e-3    # Force in Electronvolts/Angstrom

or ::

  GradAMax [Electronvolt/Angstrom] = 5.14e-3

See the manual for the list of accepted modifiers and additional convergence
criteria supported by the ``Convergence`` block.

The ``MaxSteps`` keyword specifies the maximum number of geometry optimisation
steps that the program can take before stopping, even if the specified tolerance
for the maximal force component have not been achieved by that stage of the
calculation.

Finally, the ``OutputPrefix`` keyword specifies the name of the file to be
written that will contain the present geometry during the optimisation (and then
the final geometry at the end of the calculation). The geometry is written in
gen and xyz formats to the files obtained by appending ".gen" and ".xyz"
suffixes to the specified name (`geom.out.gen` and `geom.out.xyz` in our case.)
The `dptools` package distributed with DFTB+ contains scripts (`gen2xyz` and
`xyz2gen`) to convert between the gen and the xyz formats (and various other
formats).


Hamiltonian
-----------

You have to decide upon the model used to describe your system in order to
calculate its properties. At the moment DFTB+ simplifies this decision quite a
lot, since it currently only supports types of Density Functional based Tight
Binding Hamiltonians (with some extensions). In our example, the chosen
self-consistent DFTB Hamiltonian has the following properties::

  Hamiltonian = DFTB {                 # DFTB Hamiltonian
    Scc = Yes                          # Use self-consistent charges
    SlaterKosterFiles {                # Specifying Slater-Koster files
      O-O = "../../slakos/mio-ext/O-O.skf"
      O-H = "../../slakos/mio-ext/O-H.skf"
      H-O = "../../slakos/mio-ext/H-O.skf"
      H-H = "../../slakos/mio-ext/H-H.skf"
    }
    MaxAngularMomentum {               # Maximal l-value of the various species
      O = "p"
      H = "s"
    }
  }

In this example the charge self-consistent DFTB (SCC-DFTB) method is used for
the electronic structure (and calculating the total energy, forces, etc.). This
method includes the effect of charge transfer between atoms of the system. In
order to find the final ground state of the system it has to iteratively solve
the system, until the atomic charges are self-consistently converged.
Convergence is reached if the difference between the charges used to build the
Hamiltonian and the charges obtained after the diagonalisation of the
Hamiltonian is below a certain tolerance (the default is 1e-5 electrons, but can
be tuned with the ``SccTolerance`` option). If this level of convergence is not
reached within a certain number of iterations, the code calculates the total
energy using the charges obtained so far and stops with an appropriate warning
message. The maximal number of scc-iterations is by default 100, but can be
changed via the ``MaxSccIterations`` option.

The tabulated integrals (together with other atomic and diatomic parameters)
necessary for building the DFTB Hamiltonian are stored in the so called
Slater-Koster files. Those files always describe the interaction between atom
pairs. Therefore, you have to specify, for each pairwise combination of chemical
elements in your system, the corresponding Slater-Koster file::

  SlaterKosterFiles {               # Specifying Slater-Koster files
    O-O = "../../slakos/mio-ext/O-O.skf"
    O-H = "../../slakos/mio-ext/O-H.skf"
    H-O = "../../slakos/mio-ext/H-O.skf"
    H-H = "../../slakos/mio-ext/H-H.skf"
  }

If you use a consistent file naming convention, you can avoid typing all the
file names by specifying only the generating pattern. The input::

  SlaterKosterFiles = Type2FileNames {  # File names with two atom type names
    Prefix = "../../slakos/mio-ext/"    # Prefix before first type name
    Separator = "-"                     # Dash between type names
    Suffix = ".skf"                     # Suffix after second type name
  }

would generate exactly the same file names as in the example above.

Historically the Slater-Koster file format did not contain any information about
which valence orbitals were considered when generating the interaction tables,
this can lead to data for physically inappropriate orbitals being included in
the files.  Therefore, you must provide the value of the highest orbital angular
momentum each element, specified as ``s``, ``p``, ``d`` or ``f``. This
information can be obtained from the documentation of the Slater-Koster
files. In the distributed standardised sets (available at http://www.dftb.org)
this information is contained in the documentation appended to the end of each
SK-file.

The default behaviour of the code is to assume that your system is neutral (net
electrical charge of 0). If you would like to calculate charged systems, you
have to use the ``Charge`` option. Similarly, the system is assumed to be
spin-unpolarised. You can however use the option ``SpinPolarisation`` to change
this standard behaviour.


Analysis
--------

The ``Analysis`` block contains options to calculate (or display if otherwise
only calculated internally) a number of properties. In this example, while
forces are needed to optimise the geometry, these are not usually printed in
full, only the maximum value. The ``CalculateForces`` option enables printing of
the forces.


Options
-------

The ``Options`` block contains a few global settings for the code. In the
current example, no options are specified. You could even leave out the::

  Options {}

line in the input, since the default value for the ``Options`` block is an empty
block.


ParserOptions
-------------

This block contains options which are interpreted by the parser itself and are
not passed to the main program. The most important of those options is the
``ParserVersion`` option, which tells the parser, for which version of the
parser the current input file was created for. If this is not the current parser
but an older one, the parser internally automatically converts the old input to
the new format.

The version number of the parser in the current DFTB+ code is always printed out
at the program start. It is a good habit to set this value in your input files
explicitly, like in our case::

  ParserVersion = 12

This allows you to use your input file with future versions of DFTB+ without
adapting it by hand, if the input format has changed in the more recent version.


Running DFTB+
=============

After creating the main input file, you should make sure that all the other
required files (Slater-Koster files, any files included in the HSD input via
``<<<`` constructs, etc.) are at the right place. In our example, only the
Slater-Koster files need to be present.

In order to run the calculation, you should invoke DFTB+ without any arguments
in the directory containing the file `dftb_in.hsd`. As DFTB+ writes some useful
output to the standard output (to the screen), it is recommended to save this
output for later investigation::

  dftb+ | tee output

Assuming the binary `dftb+` is in your search path, you should obtain an output
starting with::

  |===============================================================================
  |
  |  DFTB+ release 22.2
  |
  |  Copyright (C) 2006 - 2022  DFTB+ developers group
  |
  |===============================================================================
  |
  |  When publishing results obtained with DFTB+, please cite the following
  |  reference:
  |
  |  * DFTB+, a software package for efficient approximate density functional
  |    theory based atomistic simulations, J. Chem. Phys. 152, 124101 (2020).
  |    [doi: 10.1063/1.5143190]
  |
  |  You should also cite additional publications crediting the parametrization
  |  data you use. Please consult the documentation of the SK-files for the
  |  references.
  |
  |===============================================================================

  Reading input file 'dftb_in.hsd'
  Parser version: 12

  --------------------------------------------------------------------------------
  Reading SK-files:
  /home/user/slakos/mio-1-1/O-O.skf
  /home/user/slakos/mio-1-1/O-H.skf
  /home/user/slakos/mio-1-1/H-H.skf
  Done.


  Processed input in HSD format written to 'dftb_pin.hsd'

  Starting initialization...
  --------------------------------------------------------------------------------
  OpenMP threads:              16
  Chosen random seed:          1354468809
  Mode:                        Static calculation
  Self consistent charges:     Yes
  SCC-tolerance:                 0.100000E-04
  Max. scc iterations:                    100
  Shell resolved Hubbard:      No
  Spin polarisation:           No
  Nr. of up electrons:             4.000000
  Nr. of down electrons:           4.000000
  Periodic boundaries:         No
  Electronic solver:           Relatively robust
  Mixer:                       Broyden mixer
  Mixing parameter:                  0.200000
  Maximal SCC-cycles:                     100
  Nr. of chrg. vec. in memory:            100
  Electronic temperature:              0.100000E-07 H      0.272114E-06 eV
  Initial charges:             Set automatically (system chrg:   0.000E+00)
  Included shells:             O:  s, p
			       H:  s
  Extra options:
			       Mulliken analysis
			       Force calculation
  Force type                   original

  --------------------------------------------------------------------------------

  ***  Geometry step: 0

   iSCC Total electronic   Diff electronic      SCC error
      1    0.00000000E+00    0.00000000E+00    0.88081627E+00
      2   -0.39511797E+01   -0.39511797E+01    0.55742893E+00
      3   -0.39705438E+01   -0.19364070E-01    0.32497352E-01
      4   -0.39841371E+01   -0.13593374E-01    0.19288772E-02
      5   -0.39841854E+01   -0.48242063E-04    0.87062163E-05

  Total Energy:                       -3.9798793068 H         -108.2980 eV
  Extrapolated to 0K:                 -3.9798793068 H         -108.2980 eV
  Total Mermin free energy:           -3.9798793068 H         -108.2980 eV
  Force related energy:               -3.9798793068 H         -108.2980 eV

  >> Charges saved for restart in charges.bin

  total energy  -3.9798793E+00 H       energy change -3.9798793E+00 H
  gradient norm  2.3565839E-01 H/a0    max. gradient  1.8709029E-01 H/a0
  step length    0.0000000E+00 a0      max. step      0.0000000E+00 a0

  --------------------------------------------------------------------------------

  ***  Geometry step: 1

   iSCC Total electronic   Diff electronic      SCC error
      1   -0.39841856E+01    0.00000000E+00    0.84282109E-01
  .
  .
  .

If this is the case, you have managed to run DFTB+ for the first
time. Congratulations!


Examining the output
====================

DFTB+ communicates through two channels with you: by printing information to
standard output (which you should redirect into a file to keep for later
evaluation) and by writing information into various files. In the following, the
most important of these files will be introduced and analysed


Standard output
---------------

The first thing appearing in standard output after the start of DFTB+ is the
program header::

  |===============================================================================
  |
  |  DFTB+ release 22.2
  |
  |  Copyright (C) 2006 - 2022  DFTB+ developers group
  |
  |===============================================================================
  |
  |  When publishing results obtained with DFTB+, please cite the following
  |  reference:
  |
  |  * DFTB+, a software package for efficient approximate density functional
  |    theory based atomistic simulations, J. Chem. Phys. 152, 124101 (2020).
  |    [doi: 10.1063/1.5143190]
  |
  |  You should also cite additional publications crediting the parametrization
  |  data you use. Please consult the documentation of the SK-files for the
  |  references.
  |
  |===============================================================================

  Reading input file 'dftb_in.hsd'
  Parser version: 12

This tells you which program you are using (DFTB+), which release (22.2) and the
paper(s) associated with the code. Then the version of the parser used in this
DFTB+ release is listed.

As already discussed above, it can be a good habit to set this version number
explicitly in your input inside the ``ParserOptions`` block, so that::

  ParserOptions {
    ParserVersion = 12
  }

Next, the parser starts to interpret your input, then reads in the
necessary SK-files and writes the full input settings to
`dftb_pin.hsd`::

  --------------------------------------------------------------------------------
  Reading SK-files:
  /home/user/slakos/mio-1-1/O-O.skf
  /home/user/slakos/mio-1-1/O-H.skf
  /home/user/slakos/mio-1-1/H-H.skf
  Done.


  Processed input in HSD format written to 'dftb_pin.hsd'

You do not have to explicitly set all the possible options for DFTB+ in the
input, as for most of them there are default values set by the parser if not set
in the input. If you want to know which default values have been set for those
missing specifications, you should look at the processed input file
`dftb_pin.hsd`, which contains the value for all the possible input settings
(see next the subsection).

At this point the DFTB+ code is initialised and the most important parameters
of the calculation are printed out::

  Starting initialization...
  --------------------------------------------------------------------------------
  OpenMP threads:              16
  Chosen random seed:          1354468809
  Mode:                        Static calculation
  Self consistent charges:     Yes
  SCC-tolerance:                 0.100000E-04
  Max. scc iterations:                    100
  Shell resolved Hubbard:      No
  Spin polarisation:           No
  Nr. of up electrons:             4.000000
  Nr. of down electrons:           4.000000
  Periodic boundaries:         No
  Electronic solver:           Relatively robust
  Mixer:                       Broyden mixer
  Mixing parameter:                  0.200000
  Maximal SCC-cycles:                     100
  Nr. of chrg. vec. in memory:            100
  Electronic temperature:              0.100000E-07 H      0.272114E-06 eV
  Initial charges:             Set automatically (system chrg:   0.000E+00)
  Included shells:             O:  s, p
			       H:  s
  Extra options:
			       Mulliken analysis
			       Force calculation
  Force type                   original


As you can see, all quantities (e.g. electronic temperature) are converted to
the internal units of DFTB+, namely atomic units (with Hartree as the base
energy unit).

Then the program starts::

  ***  Geometry step: 0

   iSCC Total electronic   Diff electronic      SCC error
      1    0.00000000E+00    0.00000000E+00    0.88081627E+00
      2   -0.39511797E+01   -0.39511797E+01    0.55742893E+00
      3   -0.39705438E+01   -0.19364070E-01    0.32497352E-01
      4   -0.39841371E+01   -0.13593374E-01    0.19288772E-02
      5   -0.39841854E+01   -0.48242063E-04    0.87062163E-05

  Total Energy:                       -3.9798793068 H         -108.2980 eV
  Extrapolated to 0K:                 -3.9798793068 H         -108.2980 eV
  Total Mermin free energy:           -3.9798793068 H         -108.2980 eV
  Force related energy:               -3.9798793068 H         -108.2980 eV

  >> Charges saved for restart in charges.bin

  total energy  -3.9798793E+00 H       energy change -3.9798793E+00 H
  gradient norm  2.3565839E-01 H/a0    max. gradient  1.8709029E-01 H/a0
  step length    0.0000000E+00 a0      max. step      0.0000000E+00 a0
  :

Since this is an SCC calculation, DFTB+ has to iterate the charges until the
specified convergence criteria is fulfilled. In every cycle, you get information
about the values of the electronic energy, its difference to the value in the
previous SCC cycle, and the discrepancy (error) between the charges used to
build the Hamiltonian and the charges obtained after its solution. This final
value is relevant to the tolerance specified in the input (``SccTolerance``).

If the SCC cycle has converged, the total energy (including SCC and repulsive
contributions) is calculated, and similarly the total Mermin free energy (this
is the Helmholtz free energy, but where only the electronic entropy is
included). Additionally, geometry convergence relevant components are indicated.

Then the driver changes the geometry of the system, and the self-consistent
cycle is repeated as before but for the new geometry. This process continues as
long as the geometry does not converge::

  ***  Geometry step: 9

   iSCC Total electronic   Diff electronic      SCC error
      1   -0.41506534E+01    0.00000000E+00    0.33681615E-04
      2   -0.41505940E+01    0.59393461E-04    0.24963044E-04
      3   -0.41505940E+01   -0.60786931E-11    0.66000538E-11

  Total Energy:                       -4.0779379326 H         -110.9663 eV
  Extrapolated to 0K:                 -4.0779379326 H         -110.9663 eV
  Total Mermin free energy:           -4.0779379326 H         -110.9663 eV
  Force related energy:               -4.0779379326 H         -110.9663 eV

  >> Charges saved for restart in charges.bin

  total energy  -4.0779379E+00 H       energy change -2.1070335E-08 H
  gradient norm  3.0077217E-05 H/a0    max. gradient  1.9992076E-05 H/a0
  step length    1.9263985E-04 a0      max. step      1.1981494E-04 a0

  Geometry converged

If the geometry does not converge before the maximum number of geometry steps is
reached, the code will stop and you will get an appropriate warning message.
Assuming the ``MaxSteps`` option had been set to ``6`` in the input, you would
obtain::

  ***  Geometry step: 6

   iSCC Total electronic   Diff electronic      SCC error
      1   -0.41530295E+01    0.00000000E+00    0.98887987E-03
      2   -0.41529684E+01    0.61129539E-04    0.73298155E-03
      3   -0.41529684E+01   -0.52412306E-08    0.51941278E-08

  Total Energy:                       -4.0778494543 H         -110.9639 eV
  Extrapolated to 0K:                 -4.0778494543 H         -110.9639 eV
  Total Mermin free energy:           -4.0778494543 H         -110.9639 eV
  Force related energy:               -4.0778494543 H         -110.9639 eV

  >> Charges saved for restart in charges.bin

  total energy  -4.0778495E+00 H       energy change -4.9306884E-05 H
  gradient norm  6.9348797E-03 H/a0    max. gradient  4.7428785E-03 H/a0
  step length    9.5435174E-03 a0      max. step      5.5244015E-03 a0
  WARNING!
  -> !!! Geometry did NOT converge!


dftb_pin.hsd
------------

As already mentioned, the processed input file `dftb_pin.hsd` is an input file
generated from your `dftb_in.hsd` by including the default values for all
unspecified options and converting some of the input quantities to atomic
units. For example, in our case in the ``GeometryOptimization`` block several
unspecified options would appear, for which sensible default values have been
set::

  Driver = GeometryOptimization {
    Optimizer = Rational {
      DiagLimit = 1.000000000000000E-002
    }
    MovedAtoms = 1:-1
    MaxSteps = 100
    OutputPrefix = "geom.out"
    Convergence = {
      GradAMax = 1E-4
      Energy = 1.797693134862316E+308
      GradNorm = 1.797693134862316E+308
      GradElem = 1.000000000000000E-004
      DispNorm = 1.797693134862316E+308
      DispElem = 1.797693134862316E+308
    }
    LatticeOpt = No
    AppendGeometries = No
  }

Similarly, in the ``DFTB{}`` block the switch for the shell resolved SCC, for
example, has been set to the default value of ``No``::

  ShellResolvedScc = No

Options which have been explicitly set in the original input file are
unchanged. The file `dftb_pin.hsd` is itself a valid HSD input file,
and you can use it as input (after renaming it to `dftb_in.hsd`) to
re-run the calculation. It is always in the format suitable for the
current parser, even if the input in `dftb_in.hsd` was for an older
format (indicated by the appropriate ``ParserVersion``
option). Therefore, the ``ParserVersion`` option in the processed
input file `dftb_pin.hsd` is always set to the parser version
corresponding to the version of DFTB+ which generated the file.


detailed.out
------------

This file contains detailed information about the properties of your system. It
is updated continuously during the run, by the end of the calculation will
contain values calculated during the last SCC cycle. All the numerical values
given in this file are in atomic units, unless explicitly specified otherwise.

`detailed.out` contains (among other data) the number of the last geometry step
and a summary of the last SCC cycle::

  Geometry optimization step: 9


  ********************************************************************************
   iSCC Total electronic   Diff electronic      SCC error
      3   -0.41505940E+01   -0.60786931E-11    0.66000538E-11
  ********************************************************************************

Then the populaton analysis information follows::

   Total charge:    -0.00000000

   Atomic gross charges (e)
   Atom           Charge
      1      -0.59260702
      2       0.29630351
      3       0.29630351

  Nr. of electrons (up):      8.00000000
  Atom populations (up)
   Atom       Population
      1       6.59260702
      2       0.70369649
      3       0.70369649

  l-shell populations (up)
   Atom Sh.   l       Population
      1   1   0       1.73421608
      1   2   1       4.85839094
      2   1   0       0.70369649
      3   1   0       0.70369649

  Orbital populations (up)
   Atom Sh.   l   m       Population  Label
      1   1   0   0       1.73421608  s
      1   2   1  -1       1.68105131  p_y
      1   2   1   0       1.17733963  p_z
      1   2   1   1       2.00000000  p_x
      2   1   0   0       0.70369649  s
      3   1   0   0       0.70369649  s

It shows the total charge of the system and the charges for each atom, followed
by detailed population analyis for each atom, shell and orbital.

.. |H2O| replace:: H\ :sub:`2`\ O


Then you obtain a count of the total number electrons in the system, and the
number of electrons on each atom, each atomic shell of the atoms (s, p, d, etc.)
and each atomic orbital (labelled by their m\ :sub:`z` value) as calculated by
Mulliken-analysis::

  Nr. of electrons (up):      8.00000000
  Atom populations (up)
   Atom       Population
      1       6.59260702
      2       0.70369649
      3       0.70369649

  l-shell populations (up)
   Atom Sh.   l       Population
      1   1   0       1.73421608
      1   2   1       4.85839094
      2   1   0       0.70369649
      3   1   0       0.70369649

  Orbital populations (up)
   Atom Sh.   l   m       Population  Label
      1   1   0   0       1.73421608  s
      1   2   1  -1       1.68105131  p_y
      1   2   1   0       1.17733963  p_z
      1   2   1   1       2.00000000  p_x
      2   1   0   0       0.70369649  s
      3   1   0   0       0.70369649  s

In our case, due to the electronegativity difference, the hydrogen atoms are
positively charged (having only 0.704 electrons), while the oxygen atom is
negatively charged (6.59 electrons, instead of the neutral state of 6 valence
electrons).

The file then contains the Fermi energy, the different energy contributions to
the total energy and the total energy in Hartrees and electron-volts. If you are
calculating at a finite electronic temperature, you should consider using the
Mermin free energy instead of the total energy::

  Fermi level:                         0.0700493319 H            1.9061 eV
  Band energy:                        -3.6725386873 H          -99.9349 eV
  TS:                                  0.0000000000 H            0.0000 eV
  Band free energy (E-TS):            -3.6725386873 H          -99.9349 eV
  Extrapolated E(0K):                 -3.6725386873 H          -99.9349 eV
  Input / Output electrons (q):      8.0000000000      8.0000000000

  Energy H0:                          -4.1689552805 H         -113.4430 eV
  Energy SCC:                          0.0183612644 H            0.4996 eV
  Total Electronic energy:            -4.1505940161 H         -112.9434 eV
  Repulsive energy:                    0.0726560835 H            1.9771 eV
  Total energy:                       -4.0779379326 H         -110.9663 eV
  Extrapolated to 0:                  -4.0779379326 H         -110.9663 eV
  Total Mermin free energy:           -4.0779379326 H         -110.9663 eV
  Force related energy:               -4.0779379326 H         -110.9663 eV

Between the two blocks of energy data, the input and output electron numbers at
the last Hamiltonian diagonalisation are shown, so that you can check that no
electrons get lost during the calculation.

This is then followed by a confirmation that the SCC convergence has been
reached in the last geometry step::

  SCC converged

You should always make sure that this is true, so that the properties of your
system have been calculated by using convergent charges. Values obtained by
using non convergent charges are usually meaningless.

Finally you get the forces on the atoms in your system.  You get also the
maximal force component occurring in your system. After this, the dipole moment
of the system (in atomic units and Debye) is printed where possible. The end of
the file will then show whether the geometry optimisation has reached
convergence, i.e., all force components on the moved atoms are below the
specified tolerance::

  Full geometry written in geom.out.{xyz|gen}

  Total Forces
      1     -0.000000000000     -0.000008377460     -0.000000000000
      2      0.000000000000      0.000004188730      0.000019992076
      3     -0.000000000000      0.000004188730     -0.000019992076

  Maximal derivative component:        0.199921E-04 au

  Dipole moment:    0.00000000    0.64283623    0.00000000 au
  Dipole moment:    0.00000000    1.63392685    0.00000000 Debye

  Geometry converged

As indicated above, in the current case, the final relaxed geometries can be
found stored as xyz and gen format in the output files `geom.out.xyz` and
`geom.out.gen`.


band.out
--------

This file contains the energies of the individual electronic levels (orbitals)
in electronvolts, followed by the occupation of the individual single particle
levels for all of the possible spin channels. For spin unpolarised calculations
(like this one) you will get only one set of values, since the levels are spin
restricted and are twofold degenerate. In a collinear spin polarised calculation
you would obtain separate values for the spin up and spin down levels::

  KPT            1  SPIN            1  KWEIGHT    1.00000000000000
      1   -23.102  2.00000
      2   -11.275  2.00000
      3    -8.538  2.00000
      4    -7.053  2.00000
      5    10.865  0.00000
      6    15.197  0.00000

The eigenenergies are in units of electron volts. You can use the scripts
`dp_bands` in the `dptools` package to convert the data in `band.out` to
XNY-format, which can be visualised with common 2D plotting tools.

Despite its name, the file `band.out` is also created for non-periodic systems,
containing the eigenenergies and occupation numbers for molecular systems (You
should ignore the k-point index and the k-point weight in the first line in this
case).


results.tag
-----------

If you want to process the results of DFTB+ with another program, you should not
extract the information from the standard output or the human readable output
files (`detailed.out`, `band.out`, etc.), since their format could significantly
change between subsequent releases of DFTB+. By setting the ``WriteResultsTag``
to ``Yes`` in the ``Options`` block ::

  Options {
    WriteResultsTag = Yes
  }

you obtain the file `results.tag` at the end of your calculation, which contains
some of the most important data in a format easily parsed by a script or a
program. This file contains entries like::

   forces              :real:2:3,3
    -0.711965764038220E-026 -0.837746041076892E-005 -0.292432744686266E-012
     0.107287666233641E-015  0.418872998346476E-005  0.199920761760342E-004
    -0.107287666226522E-015  0.418873042729029E-005 -0.199920758836292E-004

In the first line the name of the quantity is given, followed by its type
(``real``, ``integer``, ``logical``). Then the rank of the quantity is given
(``0``: scalar, ``1``: vector, ``2``: rank 2 matrix, etc.), followed by the size
of each dimension. Following this, the data for the given quantity is dumped as
free format.


Other output files
------------------

There are also other output files not discussed in detail here. They are only
created, if appropriate choices in the ``Options`` or ``ExcitedState`` blocks
are set. Please consult the manual for further details.
