# Multivariate birth process reparameterisation of the SDE model
Simon Frost (@sdwfrost), 2020-06-12

## Introduction

[Fintzi et al.](https://arxiv.org/abs/2001.05099) reparameterise a stochastic epidemiological model in two ways:
- they consider the dynamics of time-integrated rates (infection and recovery in the SIR model); and
- they use a log-transformed scale, to model stochastic perturbations due to stochasticity on a multiplicative scale.

There are lots of advantages to this parameterisation, not the least that the states in this model more closely match the kind of data that are usually collected.

In the following, the dynamics of the cumulative numbers of infections, `C` and the number of recoveries, `R`, are explicitly modeled as `Ctilde=log(C+1)` and `Rtilde=log(R+1)`, with the dynamics of `S` and `I` determined using the initial conditions and the time-integrated rates. Although the code can be made more generic, for this tutorial, the code is kept to be specific for the SIR model for readability.

## Libraries

````julia
using DifferentialEquations
using StochasticDiffEq
using DiffEqCallbacks
using Random
using SparseArrays
using DataFrames
using StatsPlots
using BenchmarkTools
````





## Transitions

````julia
function sir_mbp!(du,u,p,t)
    (Ctilde,Rtilde) = u
    (β,c,γ,S₀,I₀,N) = p
    C = exp(Ctilde)-1.0
    R = exp(Rtilde)-1.0
    S = S₀-C
    I = I₀+C-R
    @inbounds begin
        du[1] = (exp(-Ctilde)-0.5*exp(-2.0*Ctilde))*(β*c*I/N*S)
        du[2] = (exp(-Rtilde)-0.5*exp(-2.0*Rtilde))*(γ*I)
    end
    nothing
end;
````


````
sir_mbp! (generic function with 1 method)
````





The pattern of noise for this parameterisation is a diagonal matrix.

````julia
# Define a sparse matrix by making a dense matrix and setting some values as not zero
A = zeros(2,2)
A[1,1] = 1
A[2,2] = 1
A = SparseArrays.sparse(A);
````


````
2×2 SparseMatrixCSC{Float64,Int64} with 2 stored entries:
  [1, 1]  =  1.0
  [2, 2]  =  1.0
````



````julia
# Make `g` write the sparse matrix values
function sir_noise!(du,u,p,t)
    (Ctilde,Rtilde) = u
    (β,c,γ,S₀,I₀,N) = p
    C = exp(Ctilde)-1.0
    R = exp(Rtilde)-1.0
    S = S₀-C
    I = I₀+C-R
    du[1,1] = exp(-Ctilde)*sqrt(β*c*I/N*S)
    du[2,2] = exp(-Rtilde)*sqrt(γ*I)
end;
````


````
sir_noise! (generic function with 1 method)
````





## Time domain

We set the timespan for simulations, `tspan`, initial conditions, `u0`, and parameter values, `p`, which contains both the rates of the model and the initial conditions of `S` and `I`.

````julia
δt = 0.1
tmax = 40.0
tspan = (0.0,tmax)
t = 0.0:δt:tmax;
````


````
0.0:0.1:40.0
````





## Initial conditions

````julia
u0 = [0.0,0.0]; # C,R
````


````
2-element Array{Float64,1}:
 0.0
 0.0
````





## Parameter values

````julia
p = [0.05,10.0,0.25,990.0,10.0,1000.0]; # β,c,γ,S₀,I₀,N
````


````
6-element Array{Float64,1}:
    0.05
   10.0
    0.25
  990.0
   10.0
 1000.0
````





## Random number seed

````julia
Random.seed!(1234);
````


````
MersenneTwister(UInt32[0x000004d2], Random.DSFMT.DSFMT_state(Int32[-1393240
018, 1073611148, 45497681, 1072875908, 436273599, 1073674613, -2043716458, 
1073445557, -254908435, 1072827086  …  -599655111, 1073144102, 367655457, 1
072985259, -1278750689, 1018350124, -597141475, 249849711, 382, 0]), [0.0, 
0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0  …  0.0, 0.0, 0.0, 0.0, 0.0, 0.
0, 0.0, 0.0, 0.0, 0.0], UInt128[0x00000000000000000000000000000000, 0x00000
000000000000000000000000000, 0x00000000000000000000000000000000, 0x00000000
000000000000000000000000, 0x00000000000000000000000000000000, 0x00000000000
000000000000000000000, 0x00000000000000000000000000000000, 0x00000000000000
000000000000000000, 0x00000000000000000000000000000000, 0x00000000000000000
000000000000000  …  0x00000000000000000000000000000000, 0x00000000000000000
000000000000000, 0x00000000000000000000000000000000, 0x00000000000000000000
000000000000, 0x00000000000000000000000000000000, 0x00000000000000000000000
000000000, 0x00000000000000000000000000000000, 0x00000000000000000000000000
000000, 0x00000000000000000000000000000000, 0x00000000000000000000000000000
000], 1002, 0)
````





## Defining a callback

It is possible for the number of infected individuals to become negative. Here, a simple approach is taken where the integration is stopped if the number of infected individuals becomes negative. This is implemented using a `ContinuousCallback` from the `DiffEqCallbacks` library.

````julia
function condition(u,t,integrator,p) # Event when event_f(u,t) == 0
    (Ctilde,Rtilde) = u
    (β,c,γ,S₀,I₀,N) = p
    C = exp(Ctilde)-1.0
    R = exp(Rtilde)-1.0
    S = S₀-C
    I = I₀+C-R
    I
end
````


````
condition (generic function with 1 method)
````



````julia
function affect!(integrator)
    terminate!(integrator)
end
````


````
affect! (generic function with 1 method)
````



````julia
cb = ContinuousCallback(
        (u,t,integrator)->condition(u,t,integrator,p),
        affect!)
````


````
ContinuousCallback{Main.##WeaveSandBox#359.var"#1#2",typeof(Main.##WeaveSan
dBox#359.affect!),typeof(Main.##WeaveSandBox#359.affect!),typeof(DiffEqBase
.INITIALIZE_DEFAULT),Float64,Int64,Nothing,Int64}(Main.##WeaveSandBox#359.v
ar"#1#2"(), Main.##WeaveSandBox#359.affect!, Main.##WeaveSandBox#359.affect
!, DiffEqBase.INITIALIZE_DEFAULT, nothing, true, 10, Bool[1, 1], 1, 2.22044
6049250313e-15, 0)
````





## Running the model

````julia
prob_mbp = SDEProblem(sir_mbp!,sir_noise!,u0,tspan,p,noise_rate_prototype=A)
````


````
SDEProblem with uType Array{Float64,1} and tType Float64. In-place: true
timespan: (0.0, 40.0)
u0: [0.0, 0.0]
````



````julia
sol_mbp = solve(prob_mbp,
            SRA1(),
            callback=cb,
            saveat=δt);
````


````
retcode: Success
Interpolation: 1st order linear
t: 401-element Array{Float64,1}:
  0.0
  0.1
  0.2
  0.3
  0.4
  0.5
  0.6
  0.7
  0.8
  0.9
  ⋮
 39.2
 39.3
 39.4
 39.5
 39.6
 39.7
 39.8
 39.9
 40.0
u: 401-element Array{Array{Float64,1},1}:
 [0.0, 0.0]
 [0.5560118752722868, 0.6592648794324435]
 [0.7516138822923213, 0.808165538492502]
 [0.9030334565128892, 0.8772018244479675]
 [1.5116125854694589, 0.8940295688810131]
 [1.5647181276786124, 1.0120717011063445]
 [1.815121190190538, 0.9554316727565839]
 [2.0141708060080865, 0.766427402706431]
 [2.1540530364116917, 1.0134981070768228]
 [2.2552489256888832, 1.2858319183820615]
 ⋮
 [6.692254748632999, 6.681425765334046]
 [6.6923474492825825, 6.682514787788397]
 [6.692440149932166, 6.683603810242747]
 [6.692532850581748, 6.684692832697098]
 [6.692625551231331, 6.685781855151449]
 [6.69271787932966, 6.686857498510853]
 [6.692780719392034, 6.6868741648078975]
 [6.692843559454406, 6.686890831104941]
 [6.6929063995167795, 6.686907497401985]
````





## Post-processing

We can convert the output to a dataframe for convenience.

````julia
df_mbp = DataFrame(sol_mbp(sol_mbp.t)')
df_mbp[!,:C] = exp.(df_mbp[!,:x1]) .- 1.0
df_mbp[!,:R] = exp.(df_mbp[!,:x2]) .- 1.0
df_mbp[!,:S] = p[4] .- df_mbp[!,:C]
df_mbp[!,:I] = p[5] .+ df_mbp[!,:C] .- df_mbp[!,:R]
df_mbp[!,:t] = sol_mbp.t
````


````
401-element Array{Float64,1}:
  0.0
  0.1
  0.2
  0.3
  0.4
  0.5
  0.6
  0.7
  0.8
  0.9
  ⋮
 39.2
 39.3
 39.4
 39.5
 39.6
 39.7
 39.8
 39.9
 40.0
````





## Plotting

We can now plot the results.

````julia
@df df_mbp plot(:t,
    [:S :I :R],
    label=["S" "I" "R"],
    xlabel="Time",
    ylabel="Number")
````


![](figures/mbp_15_1.png)



## Benchmarking

````julia
@benchmark solve(prob_mbp,SRA1(),callback=cb)
````


````
BenchmarkTools.Trial: 
  memory estimate:  946.47 KiB
  allocs estimate:  20762
  --------------
  minimum time:     1.601 ms (0.00% GC)
  median time:      7.764 ms (0.00% GC)
  mean time:        8.437 ms (5.77% GC)
  maximum time:     40.278 ms (70.06% GC)
  --------------
  samples:          592
  evals/sample:     1
````




## Appendix
### Computer Information
```
Julia Version 1.4.1
Commit 381693d3df* (2020-04-14 17:20 UTC)
Platform Info:
  OS: Linux (x86_64-pc-linux-gnu)
  CPU: Intel(R) Core(TM) i7-1065G7 CPU @ 1.30GHz
  WORD_SIZE: 64
  LIBM: libopenlibm
  LLVM: libLLVM-8.0.1 (ORCJIT, icelake-client)
Environment:
  JULIA_NUM_THREADS = 4

```

### Package Information

```
Status `~/.julia/environments/v1.4/Project.toml`
[46ada45e-f475-11e8-01d0-f70cc89e6671] Agents 3.1.0
[f5f396d3-230c-5e07-80e6-9fadf06146cc] ApproxBayes 0.3.2
[c52e3926-4ff0-5f6e-af25-54175e0327b1] Atom 0.12.14
[fbb218c0-5317-5bc6-957e-2ee96dd4b1f0] BSON 0.2.6
[6e4b80f9-dd63-53aa-95a3-0cdb28fa8baf] BenchmarkTools 0.5.0
[a134a8b2-14d6-55f6-9291-3336d3ab0209] BlackBoxOptim 0.5.0
[2d3116d5-4b8f-5680-861c-71f149790274] Bridge 0.11.3
[1aa9af3a-2424-508f-bb7e-0626de155470] BridgeDiffEq 0.1.0
[46d747a0-b9e1-11e9-14b5-615c73e45078] BridgeSDEInference 0.3.2
[336ed68f-0bac-5ca0-87d4-7b16caf5d00b] CSV 0.6.2
[49dc2e85-a5d0-5ad3-a950-438e2897f1b9] Calculus 0.5.1
[2445eb08-9709-466a-b3fc-47e12bd697a2] DataDrivenDiffEq 0.3.1
[a93c6f00-e57d-5684-b7b6-d8193f3e46c0] DataFrames 0.21.2
[bcd4f6db-9728-5f36-b5f7-82caef46ccdb] DelayDiffEq 5.24.1
[2b5f629d-d688-5b77-993f-72d75c75574e] DiffEqBase 6.36.4
[ebbdde9d-f333-5424-9be2-dbf1e9acfb5e] DiffEqBayes 2.15.0
[eb300fae-53e8-50a0-950c-e21f52c2b7e0] DiffEqBiological 4.3.0
[459566f4-90b8-5000-8ac3-15dfb0a30def] DiffEqCallbacks 2.13.2
[aae7a2af-3d4f-5e19-a356-7da93b79d9d0] DiffEqFlux 1.12.0
[c894b116-72e5-5b58-be3c-e6d8d4ac2b12] DiffEqJump 6.8.2
[1130ab10-4a5a-5621-a13d-e4788d82bd4c] DiffEqParamEstim 1.14.1
[41bf760c-e81c-5289-8e54-58b1f1f8abe2] DiffEqSensitivity 6.19.1
[0c46a032-eb83-5123-abaf-570d42b7fbaa] DifferentialEquations 6.14.0
[b4f34e82-e78d-54a5-968a-f98e89d6e8f7] Distances 0.8.2
[31c24e10-a181-5473-b8eb-7969acd0382f] Distributions 0.23.4
[634d3b9d-ee7a-5ddf-bec9-22491ea816e1] DrWatson 1.13.1
[587475ba-b771-5e3f-ad9e-33799f191a9c] Flux 0.10.5
[f6369f11-7733-5829-9624-2563aa707210] ForwardDiff 0.10.10
[38e38edf-8417-5370-95a0-9cbb8c7f171a] GLM 1.3.9
[28b8d3ca-fb5f-59d9-8090-bfdbd6d07a71] GR 0.49.1
[891a1506-143c-57d2-908e-e1f8e92e6de9] GaussianProcesses 0.12.0
[523d8e89-b243-5607-941c-87d699ea6713] Gillespie 0.1.0
[e850a1a4-d859-11e8-3d54-a195e6d045d3] GpABC 0.0.1
[7073ff75-c697-5162-941a-fcdaad2a7d2a] IJulia 1.21.2
[a98d9a8b-a2ab-59e6-89dd-64a1c18fca59] Interpolations 0.12.10
[4076af6c-e467-56ae-b986-b466b2749572] JuMP 0.21.2
[e5e0dc1b-0480-54bc-9374-aad01c23163d] Juno 0.8.2
[093fc24a-ae57-5d10-9952-331d41423f4d] LightGraphs 1.3.3
[c7f686f2-ff18-58e9-bc7b-31028e88f75d] MCMCChains 3.0.12
[1914dd2f-81c6-5fcd-8719-6d5c9610ff09] MacroTools 0.5.5
[5424a776-8be3-5c5b-a13f-3551f69ba0e6] Mamba 0.12.4
[ff71e718-51f3-5ec2-a782-8ffcbfa3c316] MixedModels 3.0.0-DEV
[961ee093-0014-501f-94e3-6117800e7a78] ModelingToolkit 3.8.0
[76087f3c-5699-56af-9a33-bf431cd00edd] NLopt 0.6.0
[429524aa-4258-5aef-a3af-852621145aeb] Optim 0.20.6
[1dea7af3-3e70-54e6-95c3-0bf5283fa5ed] OrdinaryDiffEq 5.39.1
[91a5bcdd-55d7-5caf-9e0b-520d859cae80] Plots 1.3.7
[c3e4b0f8-55cb-11ea-2926-15256bba5781] Pluto 0.9.4
[1a8c2f83-1ff3-5112-b086-8aa67b057ba1] Query 0.12.3-DEV
[6f49c342-dc21-5d91-9882-a32aef131414] RCall 0.13.7
[c5292f4c-5179-55e1-98c5-05642aab7184] ResumableFunctions 0.5.1
[37e2e3b7-166d-5795-8a7a-e32c996b4267] ReverseDiff 1.2.0
[428bdadb-6287-5aa5-874b-9969638295fd] SimJulia 0.8.0
[05bca326-078c-5bf0-a5bf-ce7c7982d7fd] SimpleDiffEq 1.1.0
[276daf66-3868-5448-9aa4-cd146d93841b] SpecialFunctions 0.10.3
[2913bbd2-ae8a-5f71-8c99-4fb6c76f3a91] StatsBase 0.33.0
[f3b207a7-027a-5e70-b257-86293d7955fd] StatsPlots 0.14.6
[8188c328-b5d6-583d-959b-9690869a5511] Stheno 0.6.4
[789caeaf-c7a9-5a7d-9973-96adeb23e2a0] StochasticDiffEq 6.23.1
[92b13dbe-c966-51a2-8445-caca9f8a7d42] TaylorIntegration 0.8.3
[9f7883ad-71c0-57eb-9f7f-b5c9e6d3789c] Tracker 0.2.6
[fce5fe82-541a-59a6-adf8-730c64b5f9a0] Turing 0.13.0
[1986cc42-f94f-5a68-af5c-568840ba703d] Unitful 1.2.1
[44d3d7a6-8a23-5bf8-98c5-b353f8df5ec9] Weave 0.10.2
[e88e6eb3-aa80-5325-afca-941959d7151f] Zygote 0.4.20
```