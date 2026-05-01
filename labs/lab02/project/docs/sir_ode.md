```@meta
EditURL = "../scripts/sir_ode.jl"
```

````@example sir_ode
using DrWatson
@quickactivate "project"

using DifferentialEquations
using SimpleDiffEq
using Tables
using DataFrames
using StatsPlots
using LaTeXStrings # Для красивого отображения формул на графиках
using Plots
using BenchmarkTools

script_name = splitext(basename(PROGRAM_FILE))[1]
mkpath(plotsdir(script_name))
mkpath(datadir(script_name))

function sir_ode!(du, u, p, t)
	(S, I, R) = u
	(β, c, γ) = p
	N = S + I + R
	@inbounds begin
		du[1] =-β * c * I / N * S
		du[2] = β * c * I / N * S- γ * I
		du[3] = γ * I
	end
	nothing
end
````

Параметры модели

````@example sir_ode
δt = 0.1
tmax = 40.0
tspan = (0.0, tmax)
u0 = [990.0, 10.0, 0.0] # S, I, R
p = [0.05, 10.0, 0.25] # β, c, γ
````

Расчет базового репродуктивного числа

````@example sir_ode
R0 = (p[2] * p[1]) / p[3] # R0 = (c * β) / γ
````

Создание и решение задачи

````@example sir_ode
prob_ode = ODEProblem(sir_ode!, u0, tspan, p)
sol_ode = solve(prob_ode, dt = δt)
````

Подготовка данных в DataFrame

````@example sir_ode
df_ode = DataFrame(Tables.table(sol_ode'))
rename!(df_ode, ["S", "I", "R"])
df_ode[!, :t] = sol_ode.t
df_ode[!, :N] = df_ode.S + df_ode.I + df_ode.R # Общая численность популяции
````

Вывод параметров модели

````@example sir_ode
println("Параметры модели SIR:")
println("β (вероятность заражения) = ", p[1])
println("c (среднее число контактов) = ", p[2])
println("γ (скорость выздоровления) = ", p[3])
println("R0 = c * β / γ = ", round(R0, digits=3))
println("Средняя продолжительность болезни = ", round(1/p[3], digits=2), " дней")
println("Начальные условия: S0 = ", u0[1], ", I0 = ", u0[2], ", R0 =", u0[3])
````

1. ОСНОВНОЙ ГРАФИК: динамика всех трех групп

````@example sir_ode
plt1 = @df df_ode plot(:t,
	[:S :I :R],
	label=[L"S(t)" L"I(t)" L"R(t)"],
	xlabel="Время, дни",
	ylabel="Количество людей",
	title="Модель SIR: Динамика эпидемии",
	linewidth=2,
	legend=:right,
	grid=true,
	size=(800, 500))
````

Добавление аннотаций с параметрами

````@example sir_ode
annotate!(plt1, maximum(df_ode.t) * 0.7, maximum(df_ode.N) * 0.8,
	text("Параметры:\nβ = $(p[1])\nc = $(p[2])\nγ = $(p[3])\nR0 =$(round(R0, digits=2))", 8, :left))
````

График только инфицированных (I)

````@example sir_ode
plt2 = @df df_ode plot(:t, :I,
	label=L"I(t)",
	xlabel="Время, дни",
	ylabel="Количество инфицированных",
	title="Динамика числа зараженных",
	color=:red,
	linewidth=2,
	fill=(0, 0.3, :red),
	grid=true,
	size=(800, 400))
````

Отметка пика эпидемии

````@example sir_ode
peak_idx = argmax(df_ode.I)
peak_time = df_ode.t[peak_idx]
peak_value = df_ode.I[peak_idx]
vline!(plt2, [peak_time], color=:black, linestyle=:dash, label=false, linewidth=1)
annotate!(plt2, peak_time, peak_value * 1.05,
	text("Пик: $(round(peak_value, digits=1)) на $(round(peak_time, digits=1)) день", 8, :top))
````

График в логарифмическом масштабе (для анализа экспоненциального роста)

````@example sir_ode
plt3 = @df df_ode plot(:t, :I,
	label=L"I(t)",
	xlabel="Время, дни",
	ylabel="Количество инфицированных (лог. масштаб)",
	title="Экспоненциальный рост (лог. шкала)",
	yscale=:log10,
	color=:red,
	linewidth=2,
	grid=true,
	size=(800, 400))
````

График долей населения (в процентах)

````@example sir_ode
plt4 = @df df_ode plot(:t,
	[:S :I :R] ./ df_ode.N .* 100,
	label=[L"S(t)/N" L"I(t)/N" L"R(t)/N"],
	xlabel="Время, дни",
	ylabel="Доля популяции, %",
	title="Динамика эпидемии (в процентах)",
	linewidth=2,
	legend=:right,
	grid=true,
	size=(800, 500))
````

Горизонтальная линия для порога коллективного иммунитета

````@example sir_ode
if R0 > 1
	herd_immunity_threshold = (1- 1/R0) * 100
	hline!(plt4, [herd_immunity_threshold], color=:purple, linestyle=:dash,
		label="Порог коллективного иммунитета ($(round(herd_immunity_threshold, digits=1))%)", linewidth=1.5)
end
````

Фазовый портрет (I vs S)

````@example sir_ode
plt5 = plot(df_ode.S, df_ode.I,
	label="Фазовая траектория",
	xlabel=L"S(t)",
	ylabel=L"I(t)",
	title="Фазовый портрет SIR модели",
	color=:blue,
	linewidth=2,
	grid=true,
	size=(800, 500),
	legend=:topright)
````

---

*This page was generated using [Literate.jl](https://github.com/fredrikekre/Literate.jl).*

