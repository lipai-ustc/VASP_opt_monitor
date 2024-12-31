# VASP_opt_monitor

When performing structural optimization using VASP, we typically set the convergence parameter `EDIFFG = -0.02` in the `INCAR` file. This means that the optimization will terminate when the forces on all atoms in the structure are less than 0.02 eV/Å.

The `OUTCAR` file contains information about the forces on all atoms for each ionic step, but it is not convenient to check the maximum force on atoms for each step directly from this file.

In the VASP-VTST version (capable of calculating NEB), you can use the `vef.pl` script to inspect atomic forces during structural optimization. For more details, refer to "Transition State Tools for VASP." However, this script is not applicable for non-VTST versions.

It's important to note that the force convergence threshold set by `EDIFFG` applies only to non-fixed atoms. Fixed atoms' forces are not subject to this threshold. Therefore, even if the forces have converged, the maximum force on atoms in the `OUTCAR` might still exceed the set convergence standard due to fixed atoms.

To properly check force convergence, the forces on fixed atoms should be excluded.

For ease of monitoring energy changes and the maximum force on non-fixed atoms during structural optimization, I have written the following Bash script:

```bash
#!/bin/sh
# Author: lipai@mail.ustc.edu.cn

begin=$1
if [ "$begin" == "" ]; then begin=0; fi    
awk -v begin=$begin '/E0/{if ( i<begin  ) i++;else print $0 }' OSZICAR > temp.e
    
awk '/POSITION/,/drift/{
    if(NF==6) print $4,$5,$6;
    else if($1=="total") print $1 }' OUTCAR > temp.f

awk '{if($4=="F"||$4=="T") print $4,$5,$6}' CONTCAR > temp.fix
flag=$(wc -l < temp.fix)
steps=$(grep E0 OSZICAR | tail -1 | awk '{print $1}')
if [ "$flag" != "0" ] ; then
    if [ -f temp.fixx ] ; then rm temp.fixx ; fi
    for i in $(seq $steps); do
        cat temp.fix >> temp.fixx
        echo >> temp.fixx
    done
    paste temp.f temp.fixx > temp.ff
fi

awk  '{ 
    if($1=="total") {print ++i,a;a=0}
    else {
        if($4=="F") x=0; else x=$1;
        if($5=="F") y=0; else y=$2;
        if($6=="F") z=0; else z=$3;
        force=sqrt(x^2+y^2+z^2);
        if(a<force) a=force
    }
}' temp.ff > force.conv

gnuplot <<EOF 
set term dumb
set title 'Energy of Each Ionic Step'
set xlabel 'Ionic Steps'
set ylabel 'Energy (eV)'
plot 'temp.e' u 1:5 w l t "Energy in eV"
set title 'Maximum Force of Each Ionic Step'
set xlabel 'Ionic Steps'
set ylabel 'Force (eV/Å)'
plot 'force.conv' w l t "Force in eV/Å"
EOF

if [ $steps -gt 8 ] ; then
    tail -5 force.conv > temp.fff
    tail -5 temp.e > temp.ee
    gnuplot <<EOF 
set term dumb
set title 'Energy of Last Few Ionic Steps'
set xlabel 'Ionic Steps'
set ylabel 'Energy (eV)'
plot 'temp.ee' u 1:5 w l t "Energy in eV"
set title 'Maximum Force of Last Few Ionic Steps'
set xlabel 'Ionic Steps'
set ylabel 'Force (eV/Å)'
plot 'temp.fff' w l t "Force in eV/Å"
EOF
    rm temp.fff temp.ee
fi

rm temp.e temp.f temp.ff temp.fix temp.fixx
```

Running this script in the VASP working directory will produce the following outputs:

- A plot showing the energy of each ionic step.
- A plot showing the maximum force on non-fixed atoms for each ionic step.
- If the number of steps exceeds eight, additional plots for the last few steps are provided.

This allows for easy monitoring of both energy changes and force convergence during structural optimization.

                            Energy of each ionic step

 -153 ++-***-+------+------+------+------+-----+------+------+------+-----++
      +     *+      +      +      +      +     +      +Energy in eV ****** +
 -154 ++     *                                                            ++
      |       *                                                            |
      |       *                                                            |
 -155 ++       *                                                          ++
      |        *                                                           |
 -156 ++        *                                                         ++
      |          *                                                         |
      |           *                                                        |
 -157 ++           *      **                                              ++
      |             *** **  *                                              |
 -158 ++               *     *                                            ++
      |                       *******                                      |
      |                              *****************                     |
 -159 ++                                              ******************  ++
      +      +      +      +      +      +     +      +      +      +   *  +
 -160 ++-----+------+------+------+------+-----+------+------+------+-----++
      0      2      4      6      8      10    12     14     16     18     20
                                   Ionic steps



                          Max Force of each ionic step

  12 ++-----+------+------+------+------+------+------+------+------+-----++
     +      *      +      +      +      +      Force in eV/Angstrom ****** +
     |      *      *                                                       |
  10 ++    * *     **                                                     ++
     |     * *    * *                                                      |
     |     * *    *  *                                                     |
   8 ++    *  *   *   *                                                   ++
     |    *   *   *   *                                                    |
   6 ++   *   *  *     ***                                                ++
     |    *    * *                                                         |
     |   *     * *        *                                                |
   4 ++  *     * *         *                                              ++
     |          *           *                                              |
     |          *            *  ***                                        |
   2 ++                       **   **                                     ++
     |                                    *****  ************  *****  ***  |
     +      +      +      +      +   *****     **     +      **     **     +
   0 ++-----+------+------+------+------+------+------+------+------+-----++
     0      2      4      6      8      10     12     14     16     18     20
                                   Ionic steps



                  Energy of each ionic step for the last few steps

  -159.1 ++------+--------+-------+-------+-------+--------+-------+------++
         ******  +        +       +       +       +    Energy in eV+****** +
 -159.15 ++    ***                                                        ++
         |        ***                                                      |
         |           ****                                                  |
  -159.2 ++              *                                                ++
         |                ************                                     |
 -159.25 ++                           ****                                ++
         |                                *******                          |
         |                                       ****                      |
  -159.3 ++                                          ****                 ++
         |                                               **                |
 -159.35 ++                                                ******         ++
         |                                                       ****      |
         |                                                           ****  |
  -159.4 ++                                                              **+
         +       +        +       +       +       +        +       +       *
 -159.45 ++------+--------+-------+-------+-------+--------+-------+------++
         15     15.5      16     16.5     17     17.5      18     18.5     19
                                     Ionic steps



               Max Force of each ionic step for the last few steps

  1.7 ++-------+-------+--------+--------+-------+--------+-------+-------++
      **       +       +        +        +     Force in eV/Angstrom ****** +
  1.6 ++*                                                                 ++
      |  *                                                                 |
  1.5 ++  **                                                              ++
      |     *                                                              |
  1.4 ++     *                                                            ++
      |       *                                                            |
  1.3 ++       *                        *******                           **
  1.2 ++        *                     **       ***                    ****++
      |          *                  **            ***              ***     |
  1.1 ++          **              **                 ****       ***       ++
      |             *          ***                       *  ****           |
    1 ++             *       **                           **              ++
      |               *    **                                              |
  0.9 ++                 **                                               ++
      +        +       **       +        +       +        +       +        +
  0.8 ++-------+-------+--------+--------+-------+--------+-------+-------++
      15      15.5     16      16.5      17     17.5      18     18.5      19
                                   Ionic steps
