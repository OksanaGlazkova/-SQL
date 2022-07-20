# -SQL (итоговая работа)

Описание базы данных:
 https://edu.postgrespro.ru/bookings.pdf

**Аэропорты с рейсами, выполняемые самолетом с максимальной дальностью перелета:**

_В целом, можно вывести такой запрос:_

select distinct departure_airport  
from flights f  
where aircraft_code = (  
  select aircraft_code  
from aircrafts   
order by "range" desc  
limit 1  )

_Логично, что аэропорты вылета и прилёта идентичны, т.к. самолёт прилетает в аэропорт и будет из него потом вылетать.
Но захотелось получить этому подтверждение (отмести теорию о том, что его могут перевезти из аэропорта в аэропорт), поэтому собрала другой запрос:_

select distinct arrival_airport
from flights f
where aircraft_code = (select aircraft_code 
from aircrafts 
order by "range" desc
limit 1)
union
select distinct departure_airport
from flights f1
where aircraft_code = (select aircraft_code 
from aircrafts 
order by "range" desc
limit 1)


**10 рейсов с макс.временем задержки вылета:**

select flight_no, actual_departure - scheduled_departure as departure_delay_actually
from flights
where actual_departure is not null
order by departure_delay_actually desc
limit 10

**Брони, по которым не были получены посадочные талоны:**

select t.book_ref, bp.boarding_no
from tickets t
left join boarding_passes bp on t.ticket_no = bp.ticket_no
where bp.boarding_no is null

**Количество свободных мест для каждого рейса, их % отношение к общему количеству мест в самолете:**

_Накопление пассажиров по каждому аэропорту за все время глобально в порядке идентификатора перелета:_

select f1.departure_airport, flight_id, total_boarding_no, (total_seat_no-total_boarding_no) as empty_spaces, (total_seat_no-total_boarding_no)/total_seat_no*100 as "%_empty_spaces",
	sum(total_boarding_no) over (partition by f1.departure_airport order by flight_id) as "Накопление пассажиров"
from (
		select flight_id, aircraft_code, count(boarding_no)::float as total_boarding_no, s.total_seat_no
		from boarding_passes b
		left join flights f using (flight_id)
		left join (
			select aircraft_code, count(seat_no)::float as total_seat_no
			from seats
			group by aircraft_code
			) as s using (aircraft_code)
		group by flight_id, aircraft_code, s.total_seat_no
		) as s1
left join (select departure_airport, flight_id
		from flights) as f1 using (flight_id)
order by departure_airport, flight_id

_Накопление пассажиров из каждого аэропорта на каждый день:_

with boarded as (
	select f.flight_id, f.flight_no, f.aircraft_code, f.departure_airport, f.scheduled_departure, f.actual_departure, count(bp.boarding_no) as total_boarding_no
	from flights f 
	join boarding_passes bp on bp.flight_id = f.flight_id 
	where f.actual_departure is not null
	group by f.flight_id 
),
max_seats_by_aircraft as(
	select s.aircraft_code,	count(s.seat_no) max_seats
	from seats s 
	group by s.aircraft_code 
)
select b.flight_no,	b.departure_airport, b.scheduled_departure, b.actual_departure,	b.total_boarding_no, m.max_seats - b.total_boarding_no as empty_spaces, round((m.max_seats - b.total_boarding_no) / m.max_seats :: dec, 2) * 100 "%_empty_spaces",
	sum(b.total_boarding_no) over (partition by (b.departure_airport, b.actual_departure::date) order by b.actual_departure) as "Накопление пассажиров"
from boarded b 
join max_seats_by_aircraft m on m.aircraft_code = b.aircraft_code

**Процентное соотношение перелетов по типам самолетов от общего количества.**

select aircraft_code, round(number_of_flights/(select sum(number_of_flights)::float as "result"
from (select aircraft_code, count(aircraft_code) as number_of_flights
from flights
group by aircraft_code) as fl_s)*100) as "%_flight"
from (select aircraft_code, count(aircraft_code)::float as number_of_flights
from flights
group by aircraft_code) as f_s
group by aircraft_code, number_of_flights


**Города, в которые можно  добраться бизнес - классом дешевле, чем эконом-классом в рамках перелета:**

with cte_1 as (
	select flight_id, fare_conditions, amount
	from ticket_flights
	where fare_conditions = 'Business'
	group by flight_id, fare_conditions, amount
	order by flight_id),
cte_2 as (select flight_id, fare_conditions, amount
	from ticket_flights
	where fare_conditions = 'Economy'
	group by flight_id, fare_conditions, amount
	order by flight_id)
select departure_city, arrival_city
from cte_1
join cte_2 using (flight_id)
join flights_v using (flight_id)
where cte_1.flight_id = cte_2.flight_id and cte_2.amount > cte_1.amount

**Города, между которых нет прямых рейсов:**

create materialized view task_2 as
select a.city as city1, b.city as city2
from airports a, airports b
except
select departure_city, arrival_city 
from flights_v fv

select *
from task_2
where city1 != city2
