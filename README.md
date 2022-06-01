Zaliczenie z Baz Danych i Hurtowni Danych
Robert Nowak Studia podyplomowe IDDS rok 2021/22

Została stworzona baza relacyjna w PostgreSQL przy wykorzystaniu danych ze strony:<br>
https://data.cityofchicago.org/Public-Safety/Crimes-2001-to-Present/ijzp-q8t2<br>
dotycząca przestępstw popełnianych w Chicago<br>
Zakres danych: od 01.01.2016 do 31.12.2021

Stworzone Tabele:

Cases:<br>
Case_Number[VARCHAR(10)] - Unikalny numer sprawy<br>
Date[TIMESTAMP] - Data popełnionej zbrodni/wykrozcenia w formacie rok-miesiąc-dzień<br>
Updated_On[TIMESTAMP] - Data zaaktualizowania sprawy w formacie rok-miesiąc-dzień<br>
Arrest[BOOLEAN] - Wskazuje czy doszło do aresztowania<br>
Domestic[BOOLEAN] - Wskazuje czy incydent był związany z domem<br>
Location_Description[VARCHAR(100)] - Okresla rodzaj lokacji w której doszło do zdarzenia

Crime:<br>
IUCR[VARCHAR(10)] - Jednolity kodeks zgłaszania przestępstw w stanie Illinois<br>
Primary_Type[VARCHAR(50)] - Podstawowy opis kodu IUCR<br>
Description[VARCHAR(100)] - Drugi opis kodu IUCR, podkategoria opisu podstawowego

Location:<br>
location_id[INT] - Zmienna identyficacyjna unikalna dla danego zestawu lokacji<br>
Block[VARCHAR(50)] - Częściowy adres<br>
District[INT] - Numer okręgu policji<br>
Ward[INT] - Numer oddziału policji<br>
Beat[INT] - Wskazuje "beat", w którym doszło do incydentu. "Beat" jest najmniejszym policyjnym obszarem geograficznym – każdy "beat" ma dedykowany samochód policyjny

Relacje między tabelami zostały pokazane na dołączonym rysunku.<br>
Dołączyłem również plik SQL z 5 stworzonymi widokami. Kod dla tych widoków jest poniżej.

PostgreSQL nie posiada funkcji PIVOT, zamiast niej wykorzystana została funkcja crosstab pochodząca z zewnętrznego modułu tablefunc, który można zainstalować wykonując poniższą kwerendę:<br>

CREATE EXTENSION IF NOT EXISTS tablefunc;<br>
---------------------------------------<br>
Pierwszy widok tworzy tabelę rozkładu całkowitej ilości przestępstw dla danego okręgu w przeciągu 5 lat.<br>
Rekordy z okręgiem numer 0 zostały pominięte, ponieważ 0 oznacza brak danych. Był tylko 1 taki rekord.<br>
W latach 2020 oraz 2021 kiedy wybuchła pandemia widać wyraźny spadek ilości przestępstw dla prawie wszystkic okręgów

CREATE VIEW number_crimes_district_year AS (<br>
WITH crimes_sum AS (<br>
SELECT * FROM crosstab ('<br>
SELECT district, <br>
extract("year" from date)::int AS year, <br>
COUNT(district)::int AS Number_of_crimes<br>
FROM cases c<br>
INNER JOIN location l on c.location_id = l.location_id<br>
WHERE l.district > 0<br>
GROUP BY CUBE (district, year)<br>
HAVING district IS NOT NULL<br>
AND extract("year" from date)::int IS NOT NULL<br>
ORDER BY district, year')<br>
AS numb(District INT, "2016" INT, "2017" INT, "2018" INT, <br>
                                  "2019" INT, "2020" INT, "2021" INT)<br>
    )<br>
SELECT district, "2016", "2017", "2018", "2019", "2020", "2021",<br>
SUM("2016" + "2017" + "2018" + "2019" + "2020" + "2021")::INT AS total<br>
FROM crimes_sum<br>
GROUP BY district, "2016", "2017", "2018", "2019", "2020", "2021"<br>
ORDER BY district<br>
);<br>
---------------------------------------<br>
Znalezienie okręgu z największą liczbą przestępstw, jest to okręg 11.<br>
Okrąg z najmniejszą ilością to 20 (31 zostaje pominięty przez znacząco małą ilość przestępstw, jest to mały okrąg znajdujący się wewnątrz 16, gdzie przestępstw jest więcej niż w 20)

SELECT district, total FROM number_crimes_district_year<br>
GROUP BY district, total<br>
ORDER BY total DESC;<br>
---------------------------------------<br>
Stworzenie podobnego widoku jak poprzedni, tylko dla zabójstw

CREATE VIEW number_homicide_district_year AS (<br>
WITH homicide_sum AS (<br>
SELECT * FROM crosstab ('<br>
SELECT district, <br>
extract("year" from date)::int AS year, <br>
COUNT(district)::int AS Number_of_crimes<br>
FROM cases c<br>
INNER JOIN location l on c.location_id = l.location_id<br>
INNER JOIN crime cr on c.iucr = cr.iucr<br>
WHERE l.district > 0<br>
AND cr.Primary_Type LIKE ''HOMICIDE''<br>
GROUP BY CUBE (district, year)<br>
HAVING district IS NOT NULL<br>
AND extract(''year'' from date)::int IS NOT NULL<br>
ORDER BY district, year')<br>
AS numb(District INT, "2016" INT, "2017" INT, "2018" INT, <br>
                                  "2019" INT, "2020" INT, "2021" INT)<br>
    )<br>
SELECT District, "2016", "2017", "2018", "2019", "2020", "2021",<br>
SUM("2016" + "2017" + "2018" + "2019" + "2020" + "2021")::INT AS Total<br>
FROM homicide_sum<br>
GROUP BY district, "2016", "2017", "2018", "2019", "2020", "2021"<br>
ORDER BY district<br>
);<br>
---------------------------------------<br>
Znalezienie okręgu z największą liczbą zabójstw, znów jest to okrąg 11, a najmniej ma okrąg 20

SELECT district, total FROM number_homicide_district_year<br>
GROUP BY district, total<br>
ORDER BY total DESC;<br>
---------------------------------------<br>
Porównanie ilości popełnionych konkretnych przestępstw w okręgach 11 oraz 20.<br>
W okręgu 11 widać znacznie przewarzające przestępstwa dotyczące napaści i narkotyków.

CREATE VIEW most_dangerous_least_dangerous AS (<br>
SELECT Primary_Type, <br>
COUNT(Primary_Type) FILTER (WHERE l.district = 11)::INT AS District_11,<br>
COUNT(Primary_Type) FILTER (WHERE l.district = 20)::INT AS District_20<br>
FROM cases c<br>
INNER JOIN crime cr ON c.iucr = cr.iucr<br>
INNER JOIN location l ON c.location_id = l.location_id<br>
GROUP BY Primary_Type<br>
ORDER BY District_11 DESC<br>
    );<br>
---------------------------------------<br>
W okręgu 20 jednak przeważa kradzież a przestępstw dotyczących narkotyków jest znacznie mniej

SELECT * FROM most_dangerous_least_dangerous<br>
ORDER BY District_20 DESC;<br>
---------------------------------------<br>
Widok który liczy ogólną ilość przestępstw popełnionych dla danego rodzaju lokacji z uwzględnieniem ilości aresztowań oraz policzonym procentem aresztowań do przestępstw.<br>
Należałoby zmniejszyć oraz ujednolicić dane ponieważ jest bardzo dużo lokacji z pojedyńczym przestępstwem, jednak ze względu na ograniczony czas nie jestem w stanie tego teraz zrobić.

CREATE VIEW location_desc_arrest AS (<br>
SELECT Location_Description, <br>
COUNT(Arrest) FILTER(WHERE Arrest = True)::int AS Number_of_arrests, <br>
COUNT(Location_Description)::int AS Number_of_crimes,<br>
ROUND(COUNT(Arrest) FILTER(WHERE Arrest = True)/COUNT(Location_Description)::numeric*100, 2) AS Procent_of_arrests<br>
FROM cases c<br>
GROUP BY Location_Description<br>
ORDER BY Number_of_crimes DESC<br>
    );<br>
 ---------------------------------------<br>
Ostatni widok pokazuje że miesiąc nie ma żadnego wpływu na ogólny rozkład przestępstw w Chicago.<br>
Jest równomiernie rozłożony dla każdego miesiąca na przestrzeni 5 lat.

CREATE VIEW distribution_crimes_month_year AS (<br>
WITH month_year AS (<br>
SELECT * FROM crosstab ('<br>
SELECT <br>
extract("month" from date)::INT AS month, <br>
extract("year" from date)::INT AS year, <br>
COUNT(district)::int AS Number_of_crimes<br>
FROM cases c<br>
INNER JOIN location l on c.location_id = l.location_id<br>
GROUP BY CUBE (month, year)<br>
HAVING extract("month" from date)::INT IS NOT NULL<br>
AND extract("year" from date)::int IS NOT NULL<br>
ORDER BY month, year')<br>
AS numb(month INT, "2016" INT, "2017" INT, "2018" INT, <br>
                                  "2019" INT, "2020" INT, "2021" INT)<br>
    )<br>
SELECT month, "2016", "2017", "2018", "2019", "2020", "2021",<br>
SUM("2016" + "2017" + "2018" + "2019" + "2020" + "2021")::INT AS Total<br>
FROM month_year<br>
GROUP BY month, "2016", "2017", "2018", "2019", "2020", "2021"<br>
ORDER BY month<br>
    );
