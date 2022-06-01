/*
Robert Nowak Studia podyplomowe IDDS rok 2021/22
Zaliczenie z Baz Danych i Hurtowni Danych

Została stworzona baza relacyjna w PostgreSQL przy wykorzystaniu danych ze strony:
https://data.cityofchicago.org/Public-Safety/Crimes-2001-to-Present/ijzp-q8t2
dotycząca przestępstw popełnianych w Chicago
Zakres danych: od 01.01.2016 do 31.12.2021

Stworzone Tabele:

Cases:
Case_Number[VARCHAR(10)] - Unikalny numer sprawy
Date[TIMESTAMP] - Data popełnionej zbrodni/wykrozcenia w formacie rok-miesiąc-dzień
Updated_On[TIMESTAMP] - Data zaaktualizowania sprawy w formacie rok-miesiąc-dzień
Arrest[BOOLEAN] - Wskazuje czy doszło do aresztowania
Domestic[BOOLEAN] - Wskazuje czy incydent był związany z domem
Location_Description[VARCHAR(100)] - Okresla rodzaj lokacji w której doszło do zdarzenia

Crime:
IUCR[VARCHAR(10)] - Jednolity kodeks zgłaszania przestępstw w stanie Illinois
Primary_Type[VARCHAR(50)] - Podstawowy opis kodu IUCR
Description[VARCHAR(100)] - Drugi opis kodu IUCR, podkategoria opisu podstawowego

Location:
location_id[INT] - Zmienna identyficacyjna unikalna dla danego zestawu lokacji
Block[VARCHAR(50)] - Częściowy adres
District[INT] - Numer okręgu policji
Ward[INT] - Numer oddziału policji
Beat[INT] - Wskazuje "beat", w którym doszło do incydentu. "Beat" jest najmniejszym policyjnym obszarem geograficznym – każdy "beat" ma dedykowany samochód policyjny

Relacje między tabelami zostały pokazane na dołączonym rysunku.
Dołączyłem również plik SQL z 5 stworzonymi widokami. Kod dla tych widoków jest poniżej.

PostgreSQL nie posiada funkcji PIVOT, zamiast niej wykorzystana została funkcja crosstab pochodząca z zewnętrznego modułu tablefunc, który można zainstalować wykonując poniższą kwerendę:
*/
CREATE EXTENSION IF NOT EXISTS tablefunc;

/*
Pierwszy widok tworzy tabelę rozkładu całkowitej ilości przestępstw dla danego okręgu w przeciągu 5 lat.
Rekordy z okręgiem numer 0 zostały pominięte, ponieważ 0 oznacza brak danych. Był tylko 1 taki rekord.
W latach 2020 oraz 2021 kiedy wybuchła pandemia widać wyraźny spadek ilości przestępstw dla prawie wszystkic okręgów
*/

CREATE VIEW number_crimes_district_year AS (
WITH crimes_sum AS (
SELECT * FROM crosstab ('
SELECT district, 
extract("year" from date)::int AS year, 
COUNT(district)::int AS Number_of_crimes
FROM cases c
INNER JOIN location l on c.location_id = l.location_id
WHERE l.district > 0
GROUP BY CUBE (district, year)
HAVING district IS NOT NULL
AND extract("year" from date)::int IS NOT NULL
ORDER BY district, year')
AS numb(District INT, "2016" INT, "2017" INT, "2018" INT, 
                                  "2019" INT, "2020" INT, "2021" INT)
    )
SELECT district, "2016", "2017", "2018", "2019", "2020", "2021",
SUM("2016" + "2017" + "2018" + "2019" + "2020" + "2021")::INT AS total
FROM crimes_sum
GROUP BY district, "2016", "2017", "2018", "2019", "2020", "2021"
ORDER BY district
);

/*
Znalezienie okręgu z największą liczbą przestępstw, jest to okręg 11
Okrąg z najmniejszą ilością to 20 (31 zostaje pominięty przez znacząco małą ilość przestępstw, jest to mały okrąg znajdujący się wewnątrz 16, gdzie przestępstw jest więcej niż w 20)
*/
SELECT district, total FROM number_crimes_district_year
GROUP BY district, total
ORDER BY total DESC;

--Stworzenie podobnego widoku jak poprzedni, tylko dla zabójstw
CREATE VIEW number_homicide_district_year AS (
WITH homicide_sum AS (
SELECT * FROM crosstab ('
SELECT district, 
extract("year" from date)::int AS year, 
COUNT(district)::int AS Number_of_crimes
FROM cases c
INNER JOIN location l on c.location_id = l.location_id
INNER JOIN crime cr on c.iucr = cr.iucr
WHERE l.district > 0
AND cr.Primary_Type LIKE ''HOMICIDE''
GROUP BY CUBE (district, year)
HAVING district IS NOT NULL
AND extract(''year'' from date)::int IS NOT NULL
ORDER BY district, year')
AS numb(District INT, "2016" INT, "2017" INT, "2018" INT, 
                                  "2019" INT, "2020" INT, "2021" INT)
    )
SELECT District, "2016", "2017", "2018", "2019", "2020", "2021",
SUM("2016" + "2017" + "2018" + "2019" + "2020" + "2021")::INT AS Total
FROM homicide_sum
GROUP BY district, "2016", "2017", "2018", "2019", "2020", "2021"
ORDER BY district
);

--Znalezienie okręgu z największą liczbą zabójstw, znów jest to okrąg 11, a najmniej ma okrąg 20
SELECT district, total FROM number_homicide_district_year
GROUP BY district, total
ORDER BY total DESC;

/*
Porównanie ilości popełnionych konkretnych przestępstw w okręgach 11 oraz 20.
W okręgu 11 widać znacznie przewarzające przestępstwa dotyczące napaści i narkotyków
*/

CREATE VIEW most_dangerous_least_dangerous AS (
SELECT Primary_Type, 
COUNT(Primary_Type) FILTER (WHERE l.district = 11)::INT AS District_11,
COUNT(Primary_Type) FILTER (WHERE l.district = 20)::INT AS District_20
FROM cases c
INNER JOIN crime cr ON c.iucr = cr.iucr
INNER JOIN location l ON c.location_id = l.location_id
GROUP BY Primary_Type
ORDER BY District_11 DESC
    );
    
/*
W okręgu 20 jednak przeważa kradzież a przestępstw dotyczących narkotyków jest znacznie mniej
*/
SELECT * FROM most_dangerous_least_dangerous
ORDER BY District_20 DESC;

/*
Widok który liczy ogólną ilość przestępstw popełnionych dla danego rodzaju lokacji z uwzględnieniem ilości aresztowań oraz policzonym procentem aresztowań do przestępstw.
Należałoby zmniejszyć oraz ujednolicić dane ponieważ jest bardzo dużo lokacji z pojedyńczym przestępstwem, jednak ze względu na ograniczony czas nie jestem w stanie tego teraz zrobić.
*/
CREATE VIEW location_desc_arrest AS (
SELECT Location_Description, 
COUNT(Arrest) FILTER(WHERE Arrest = True)::int AS Number_of_arrests, 
COUNT(Location_Description)::int AS Number_of_crimes,
ROUND(COUNT(Arrest) FILTER(WHERE Arrest = True)/COUNT(Location_Description)::numeric*100, 2) AS Procent_of_arrests
FROM cases c
GROUP BY Location_Description
ORDER BY Number_of_crimes DESC
    );
    
/*
Ostatni widok pokazuje że miesiąc nie ma żadnego wpływu na ogólny rozkład przestępstw w Chicago.
Jest równomiernie rozłożony dla każdego miesiąca na przestrzeni 5 lat.
*/
CREATE VIEW distribution_crimes_month_year AS (
WITH month_year AS (
SELECT * FROM crosstab ('
SELECT 
extract("month" from date)::INT AS month, 
extract("year" from date)::INT AS year, 
COUNT(district)::int AS Number_of_crimes
FROM cases c
INNER JOIN location l on c.location_id = l.location_id
GROUP BY CUBE (month, year)
HAVING extract("month" from date)::INT IS NOT NULL
AND extract("year" from date)::int IS NOT NULL
ORDER BY month, year')
AS numb(month INT, "2016" INT, "2017" INT, "2018" INT, 
                                  "2019" INT, "2020" INT, "2021" INT)
    )
SELECT month, "2016", "2017", "2018", "2019", "2020", "2021",
SUM("2016" + "2017" + "2018" + "2019" + "2020" + "2021")::INT AS Total
FROM month_year
GROUP BY month, "2016", "2017", "2018", "2019", "2020", "2021"
ORDER BY month
    );
