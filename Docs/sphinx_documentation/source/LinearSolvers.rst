.. role:: cpp(code)
   :language: c++

.. role:: fortran(code)
   :language: fortran


MLMG and Linear Operator Classes
================================

``MLMG`` is a class for solving the linear system using the geometric
multigrid method.  The constructor of ``MLMG`` takes the reference to
:cpp:`MLLinOp`, an abstract base class of various linear operator
classes, :cpp:`MLABecLaplacian`, :cpp:`MLPoisson`,
:cpp:`MLNodeLaplacian`, etc.  We choose the type of linear operator
class according to the type the linear system to solve.

- :cpp:`MLABecLaplacian` for cell-centered canonical form (equation :eq:`eqn::abeclap`).

- :cpp:`MLPoisson` for cell-centered constant coefficient Poisson's
  equation :math:`\nabla^2 \phi = f`.

- :cpp:`MLNodeLaplacian` for nodal variable coefficient Poisson's
  equation :math:`\nabla \cdot (\sigma \nabla \phi) = f`.

The constructors of these linear operator classes are in the form like
below

.. highlight:: c++

::

    MLABecLaplacian (const Vector<Geometry>& a_geom,
                     const Vector<BoxArray>& a_grids,
                     const Vector<DistributionMapping>& a_dmap,
                     const LPInfo& a_info = LPInfo(),
                     const Vector<FabFactory<FArrayBox> const*>& a_factory = {});

It takes :cpp:`Vectors` of :cpp:`Geometry`, :cpp:`BoxArray` and
:cpp:`DistributionMapping`.  The arguments are :cpp:`Vectors` because MLMG can
do multi-level composite solve.  If you are using it for single-level,
you can do

.. highlight:: c++

::

    // Given Geometry geom, BoxArray grids, and DistributionMapping dmap on single level
    MLABecLaplacian mlabeclaplacian({geom}, {grids}, {dmap});

to let the compiler construct :cpp:`Vectors` for you.  Recall that the
classes :cpp:`Vector`, :cpp:`Geometry`, :cpp:`BoxArray`, and
:cpp:`DistributionMapping` are defined in chapter :ref:`Chap:Basics`.  There are
two new classes that are optional parameters.  :cpp:`LPInfo` is a
class for passing parameters.  :cpp:`FabFactory` is used in problems
with embedded boundaries (chapter :ref:`Chap:EB`).

After the linear operator is built, we need to set up boundary
conditions.  This will be discussed later in section
:ref:`sec:linearsolver:bc`.

For :cpp:`MLABecLaplacian`, we next need to call member functions

.. highlight:: c++

::

    void setScalars (Real alpha, Real beta);
    void setACoeffs (int amrlev, const MultiFab& A);
    void setBCoeffs (int amrlev, const Array<MultiFab const*,AMREX_SPACEDIM>& B);

to set up the coefficients for equation :eq:`eqn::abeclap`. This is unecessary for
:cpp:`MLPoisson`, as there are no coefficients to set.  For :cpp:`MLNodeLaplacian`,
one needs to call the member function

.. highlight:: c++

::

    void setSigma (int amrlev, const MultiFab& a_sigma);

The :cpp:`int amrlev` parameter should be zero for single-level
solves.  For multi-level solves, each level needs to be provided with
``A`` and ``B``, or ``Sigma``.  For composite solves, :cpp:`amrlev` 0 will
mean the lowest level for the solver, which is not necessarily the lowest
level in the AMR hierarchy. This is so solves can be done on different sections
of the AMR hierarchy, e.g. on AMR levels 3 to 5.

After boundary conditions and coefficients are prescribed, the linear
operator is ready for an MLMG object like below.

.. highlight:: C++

::

    MLMG mlmg(mlabeclaplacian);

Optional parameters can be set (see section :ref:`sec:linearsolver:pars`),
and then we can use the ``MLMG`` member function

.. highlight:: C++

::

    Real solve (const Vector<MultiFab*>& a_sol,
                const Vector<MultiFab const*>& a_rhs,
                Real a_tol_rel, Real a_tol_abs);

to solve the problem given an initial guess and a right-hand side.
Zero is a perfectly fine initial guess.  The two :cpp:`Reals` in the argument
list are the targeted relative and absolute error tolerances.
The solver will terminate when one of these targets is met.
Set the absolute tolerance to zero if one
does not have a good value for it.  The return value of :cpp:`solve`
is the max-norm error.

After the solver returns successfully, if needed, we can call

.. highlight:: c++

::

    void compResidual (const Vector<MultiFab*>& a_res,
                       const Vector<MultiFab*>& a_sol,
                       const Vector<MultiFab const*>& a_rhs);

to compute residual (i.e., :math:`f - L(\phi)`) given the solution and
the right-hand side.  For cell-centered solvers, we can also call the
following functions to compute gradient :math:`\nabla \phi` and fluxes
:math:`-B \nabla \phi`.

.. highlight:: c++

::

    void getGradSolution (const Vector<Array<MultiFab*,AMREX_SPACEDIM> >& a_grad_sol);
    void getFluxes       (const Vector<Array<MultiFab*,AMREX_SPACEDIM> >& a_fluxes);


.. _sec:linearsolver:bc:

Boundary Conditions
===================

We now discuss how to set up boundary conditions for linear operators.
In the following, physical domain boundaries refer to the boundaries
of the physical domain, whereas coarse/fine boundaries refer to the
boundaries between AMR levels. The following steps must be
followed in the exact order.

First we need to set physical domain boundary types via :cpp:`MLLinOp` member
function

.. highlight:: c++

::

    void setDomainBC (const Array<BCType,AMREX_SPACEDIM>& lobc,  // for lower ends
                      const Array<BCType,AMREX_SPACEDIM>& hibc); // for higher ends

The supported BC types at
the physical domain boundaries are

- :cpp:`LinOpBCType::Periodic` for periodic boundary.

- :cpp:`LinOpBCType::Dirichlet` for Dirichlet boundary condition.

- :cpp:`LinOpBCType::Neumann` for homogeneous Neumann boundary condition.

- :cpp:`LinOpBCType::reflect_odd` for reflection with sign changed.

The second step is to set up coarse/fine Dirichlet boundary
conditions.  This step is not always needed.  But when it is, it
must be completed before step 3. This step is not needed when the
coarsest level in the solver covers the whole computational domain
(e.g., the coarsest level is AMR level 0).  Note that this step is
still needed in the case that the solver is used to do a single-level
solve on a fine AMR level not covering the whole domain.  The
:cpp:`MLLinOp` member function for this step is

.. highlight:: c++

::

    void setCoarseFineBC (const MultiFab* crse, int crse_ratio);

Here :cpp:`const MultiFab* crse` contains the Dirichlet boundary
values at the coarse resolution, and :cpp:`int crse_ratio` (e.g., 2)
is the refinement ratio between the coarsest solver level and the AMR
level below it.

In the third step, for each level, we call :cpp:`MLLinOp` member
function

.. highlight:: c++

::

    virtual void setLevelBC (int amrlev, const MultiFab* levelbcdata) = 0;

to set up Dirichlet boundary values.  Here the ``MultiFab`` must have
one ghost cell.  Although ``MultiFab`` is used to pass the data, only
the values in the ghost cells at Dirichlet boundaries are used.  If
there are no Dirichlet boundaries, we still need to make this call,
but we could call it with :cpp:`nullptr`.  It should be emphasized
that the data in ``levelbcdata`` for Dirichlet boundaries are assumed
to be exactly on the face of the physical domain even though for
cell-centered solvers the sought unknowns are at cell centers.  And
for cell-centered solvers, the ``MultiFab`` argument must have the
cell-centered type.

.. _sec:linearsolver:pars:

Parameters
==========

There are many parameters that can be set.  Here we discuss some
commonly used ones.

:cpp:`MLLinOp::setVerbose(int)`, :cpp:`MLMG::setVerbose(int)` and
:cpp:`MLMG:setBottomVerbose(int)` can be control the verbosity of the
linear operator, multigrid solver and the bottom solver, respectively.

The multigrid solver is an iterative solver.  The maximal number of
iterations can be changed with :cpp:`MLMG::setMaxIter(int)`.  We can
also do a fixed number of iterations with
:cpp:`MLMG::setFixedIter(int)`.  By default, V-cycle is used.  We can
use :cpp:`MLMG::setMaxFmgIter(int)` to control how many full multigrid
cycles can be done before switching to V-cycle.

:cpp:`LPInfo::setMaxCoarseningLevel(int)` can be used to control the
maximal number of multigrid levels.  We usually should not call this
function.  However, we sometimes build the solver to simply apply the
operator (e.g., :math:`L(\phi)`) without needing to solve the system.
We can do something as follows to avoid the cost of building coarsened
operators for the multigrid.

.. highlight:: c++

::

    MLABecLaplacian mlabeclap({geom}, {grids}, {dmap}, LPInfo().setMaxCoarseningLevel(0));
    // set up BC
    // set up coefficients
    MLMG mlmg(mlabeclap);
    // out = L(in)
    mlmg.apply(out, in);  // here both in and out are const Vector<MultiFab*>&

At the bottom of the multigrid cycles, we use the biconjugate gradient
stabilized method as the bottom solver.  :cpp:`MLMG` member method

.. highlight:: c++

::

    void setBottomSolver (BottomSolver s);

can be used to change the bottom solver.  Available choices are

- :cpp:`MLMG::BottomSolver::smoother`: Smoother such as Gauss-Seidel.

- :cpp:`MLMG::BottomSolver::bicgstab`: The default.

- :cpp:`MLMG::BottomSolver::cg`: The conjugate gradient method.  The
  matrix must be symmetric.

- :cpp:`MLMG::BottomSolver::Hypre`: BoomerAMG in hypre.  Currently for
  cell-centered only.

Curvilinear Coordinates
=======================

The linear solvers support curvilinear coordinates including 1D
spherical and 2d cylindrical :math:`(r,z)`.  In those cases, the
divergence operator has extra metric terms.  If one does not want the
solver to include the metric terms because they have been handled in
other ways, one can call :cpp:`setMetricTerm(bool)` with :cpp:`false`
on the :cpp:`LPInfo` object passed to the constructor of linear
operators.

hypre
=====

AMReX can use hypre BoomerAMG as a bottom solver (currently for
cell-centered problems only), as we have mentioned.  For challenging
problems, our geometric multigrid solver may have difficulty solving,
whereas an algebraic multigrid method might be more robust.  We note
that by default our solver always tries to geometrically coarsen the
problem as much as possible.  However, as we have mentioned, we can
call :cpp:`setMaxCoarseningLevel(0)` on the :cpp:`LPInfo` object
passed to the constructor of a linear operator to disable the
coarsening completely.  In that case the bottom solver is solving the
residual correction form of the original problem.  To use hypre, one
must include ``amrex/Src/Extern/HYPRE`` in the build system. For an
example of using hypre, we refer the reader to
``Tutorials/LinearSolvers/ABecLaplacian_C``.

MAC Projection
=========================

Some codes define a velocity field :math:`U = (u,v,w)` on faces, i.e. 
:math:`u` is defined on x-faces, :math:`v` is defined on y-faces,
and :math:`w` is defined on z-faces.   We refer to the exact projection 
of this velocity field as a MAC projection, in which we solve 

.. math::

   D( B \nabla \phi) = D(U^*) 

then set 

.. math::

   U = U^* - B \nabla \phi


where :math:`U^*` is a vector field (typically velocity) that we want to make divergence-free.

The MACProjection class can be defined and used to perform the MAC projection with explcitly
calling the solver directly.  In addition to solving the variable coefficient Poisson equation,
the MacProjector first computes the divergence of the vector field ( :math:`D(U^*) 
to compute a right-hand-side, and after the solve, subtracts the weighted gradient term to
make the vector field result divergence-free.  Note that in the simplest case, 
both the right-hand-side array and solution array are internal to the MacProjector object;
the user does not allocate those arrays and does not have access to them.

There is an alternative call -- commented out below -- in which a user does allocate 
the solution array and passes it in/out.

The following code is taken from 
``Tutorials/LinearOperator/MAC_Projection_EB/main.cpp`` and demonstrates how to set up 
the MACProjector object and use it to perform a MAC projection.

.. highlight:: c++

::

    // This object provides access to the EB database in the format of basic AMReX objects
    // such as BaseFab, FArrayBox, FabArray, and MultiFab
    EBFArrayBoxFactory factory(eb_level, geom, grids, dmap, ng_ebs, ebs);

    for (int idim = 0; idim < AMREX_SPACEDIM; ++idim) {
        vel[idim].define (amrex::convert(grids,IntVect::TheDimensionVector(idim)), dmap, 1, 1, MFInfo(), factory);
        beta[idim].define(amrex::convert(grids,IntVect::TheDimensionVector(idim)), dmap, 1, 0, MFInfo(), factory);
        beta[idim].setVal(1.0);
    }

    // If we want to use phi elsewhere, we must create an array in which to return the solution 
    // phi_inout.define(grids, dmap, 1, 1, MFInfo(), factory);

    // set initial velocity to U=(1,0,0)
    AMREX_D_TERM(vel[0].setVal(1.0);,
                 vel[1].setVal(0.0);,
                 vel[2].setVal(0.0););

    LPInfo lp_info;

    // If we want to use hypre to solve the full problem we do not need to coarsen the GMG stencils
    if (use_hypre) 
        lp_info.setMaxCoarseningLevel(0);

    MacProjector macproj({amrex::GetArrOfPtrs(vel)},       // face-based velocity
                         {amrex::GetArrOfConstPtrs(beta)}, // beta
                         {geom},                           // the geometry object
                         lp_info);                         // structure for passing info to the operator

    // Set bottom-solver to use hypre instead of native BiCGStab 
    if (use_hypre) 
       macproj.setBottomSolver(MLMG::BottomSolver::hypre);

    // Hard-wire the boundary conditions to be Neumann on the low x-face, Dirichlet on the high x-face,
    //  and periodic in the other two directions  
    //  (the first argument is for the low end, the second is for the high end)
    macproj.setDomainBC({AMREX_D_DECL(LinOpBCType::Neumann,
                                      LinOpBCType::Periodic,
                                      LinOpBCType::Periodic)},
                        {AMREX_D_DECL(LinOpBCType::Dirichlet,
                                      LinOpBCType::Periodic,
                                      LinOpBCType::Periodic)});

    macproj.setVerbose(mg_verbose);
    macproj.setCGVerbose(cg_verbose);

    // Define the relative tolerance
    Real reltol = 1.e-8;

    // Define the absolute tolerance; note that this argument is optional
    Real abstol = 1.e-15;

    // Solve for phi and subtract from the velocity to make it divergence-free
    macproj.project(reltol,abstol);

    // If we want to use phi elsewhere, we can pass in an array in which to return the solution 
    // macproj.project(phi_inout,reltol,abstol);


See ``Tutorials/LinearOperator/MAC_Projection_EB`` for the complete working example.


Multi-Component Operators
=========================

This section discusses solving linear systems in which the solution variable :math:`\mathbf{\phi}` has multiple components.
An example (implemented in the ``MultiComponent`` tutorial) might be:

.. math::

   D(\mathbf{\phi})_i = \sum_{i=1}^N \alpha_{ij} \nabla^2 \phi_j

(Note: only operators of the form :math:`D:\mathbb{R}^n\to\mathbb{R}^n` are currently allowed.)

- To implement a multi-component *cell-based* operator, inherit from the ``MLCellLinOp`` class.
  Override the ``getNComp`` function to return the number of components (``N``)that the operator will use.
  The solution and rhs fabs must also have at least one ghost node.
  ``Fapply``, ``Fsmooth``, ``Fflux`` must be implemented such that the solution and rhs fabs all have ``N`` components.

- Implementing a multi-component *node-based* operator is slightly different.
  A MC nodal operator must specify that the reflux-free coarse/fine strategy is being used by the solver.

  .. code::

     solver.setCFStrategy(MLMG::CFStrategy::ghostnodes);

  The reflux-free method circumvents the need to implement a special ``reflux`` at the coarse-fine boundary.
  This is accomplished by using ghost nodes.
  Each AMR level must have 2 layers of ghost nodes.
  The second (outermost) layer of nodes is treated as constant by the relaxation, essentially acting as a Dirichlet boundary.
  The first layer of nodes is evolved using the relaxation, in the same manner as the rest of the solution.
  When the residual is restricted onto the coarse level (in ``reflux``) this allows the residual at the coarse-fine boundary to be interpolated using the first layer of ghost nodes.
  :numref:`fig::refluxfreecoarsefine` illustrates the how the coarse-fine update takes place.

  .. _fig::refluxfreecoarsefine:

  .. figure:: ./LinearSolvers/refluxfreecoarsefine.png
	      :height: 2cm
	      :align: center

	      : Reflux-free coarse-fine boundary update.
	      Level 2 ghost nodes (small dark blue) are interpolated from coarse boundary.
	      Level 1 ghost nodes are updated during the relaxation along with all the other interior fine nodes.
	      Coarse nodes (large blue) on the coarse/fine boundary are updated by restricting with interior nodes
	      and the first level of ghost nodes.
	      Coarse nodes underneath level 2 ghost nodes are not updated.
	      The remaining coarse nodes are updates by restriction.
	      
  The MC nodal operator can inherit from the ``MCNodeLinOp`` class.
  ``Fapply``, ``Fsmooth``, and ``Fflux`` must update level 1 ghost nodes that are inside the domain.
  `interpolation` and `restriction` can be implemented as usual.
  `reflux` is a straightforward restriction from fine to coarse, using level 1 ghost nodes for restriction as described above.
  
  See ``Tutorials/LinearOperator/MultiComponent`` for a complete working example.

   

.. solver reuse

