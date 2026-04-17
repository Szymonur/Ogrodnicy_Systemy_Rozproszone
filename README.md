# Ogrodnicy_Systemy_Rozproszone

## **Opis poblemu:**

Trudno o dostęp do dzikiej natury w wielkich miastach, zamiast tego więc mieszkańcy pewnego wieżowca zorganizowali sobie
szklarnię na szczycie budynku. Co jakiś czas teraz Ogrodnicy ruszają do szklarni, by popielić swoje tycie ogródki. Ponieważ
ogródki są naprawdę tycie, do ich pielenia trzeba tycich grabek, maluśkich łopaciątek i maciupeńkich dawek nawozów. Dlatego
właśnie Ogrodnicy muszą najpierw w mobilnej aplikacji zarezerwować dostęp do jednej z G grabek, L łopaciątek i jednej dawki nawozu.
Relacje między G a L są nieznane (nie wiadomo, czy więcej jest łopaciątek, czy grabek).
Dodatkowo Ogrodnicy wyznaczyli jednego z Ogrodników na kupca nawozów. Grabki i łopaciątka się nie zużywają, ale nawozy i owszem.
Gdy zabraknie nawozu, wyznaczony Ogrodnik-Kupiec musi zostać powiadomiony przez innych Ogrodników, ruszyć na zakup i uzupełnić
zapasy do maksymalnej wielkości D.
Ogrodnik-Kupiec może też sam spostrzec brak nawozu; może go powiadomić jakiś jeden Ogrodnik.
Wersja uproszczona: Nie ma nawozów, tylko grabki i łopaciątka

## **Rozwiązanie:**

#### \*W celu ograniczenia ilości komunikatów i wymaganej pamięci możemy uznać, że Grabki i Łopaciątka występują w zestawach \- zestawów jest tyle ile MIN(G, L) \- wtedy traktujemy zestawy jako jeden zasób i wymagamy utrzymywania tylko jednej kolejki żądań dla tego zasobu.

### **Inicjalizacja**

- Każdy proces (Ogrodnik) zna stałe G (liczba grabek), L (liczba łopatek), D (maksymalna pojemność nawozu) oraz N (liczba procesów-ogrodników)
- Każdy proces utrzymuje 3 (\*2) lokalne kolejki żądań (dla grabek, łopatek, nawozu) posortowane po zegarze Lamporta.
- Każdy proces utrzymuje lokalny licznik aktualnie dostępnego nawozu (startowo równy D).
- Każdy proces utrzymuje wektor długości N (zainicjalizowany zerami) przechowujący na k-tej pozycji ostatnią znaną wartość zegaru Lamporta procesu k-tego.

### **Typy wiadomości:**

#### Każda wiadomość niesie ze sobą informację o tym kto jest jej nadawcą (id_procesu_wysyłającego) oraz zegar wektorowy nadawcy w momencie wysłania (v).

- REQ \- prośba o zasoby. Proces wysyłający (proces i-ty) dołącza do niej wektor (v) jego aktualnego zegara wektorowego \- wartość v\[i\] stanowi priorytet żądania.
- ACK \- potwierdzenie otrzymania prośby wysyłane w odpowiedzi na REQ. Proces wysyłający dołącza do niej wektor (v) jego aktualnego zegara wektorowego.
- RELEASE(typ_zasobu) \- informuje o zwolnieniu danego (typ_zasobu) przez proces wysyłający komunikat.
- DELIVERY \- informuje, że liczba nawozu w magazynie zwiększyła się o D

### **Odbieranie wiadomości:**

#### Przy każdym odebraniu komunikatu, proces odbierający aktualizuje swój zegar wektorowy, wg. Algorytmu Matterna

#### **Sortowanie kolejek**

Każda kolejka jest sortowana rosnąco według następujących parametrów:

1. Wartość priorytetu żądania (jeśli są równe, wtedy 2.)
2. Unikalne id procesu-właściciela żądania

#### **Kontrola nad własnymi REQ:**

1. Proces i-ty zwiększa v_w\[i\] o 1
2. Proces i-ty wysyła REQ z własnym zegarem (v_w)
3. Proces i-ty oczekuje na komunikaty zwrotne ACK od N-1 unikalnych procesów
    1. komunikat ACK nie zostaje uznany, jeśli dołączone do niego v_obcy spełnia warunek: v_obcy\[i\] \< v_w\[i\] (zapewnia aby zegar odpowiedzi był świeższy niż zegar nadania)
4. Proces i-ty wstawia swoje REQ do swojej lokalnej kolejki żądań dla każdego zasobu zgodnie z **sortowaniem kolejek**
5. Sprawdzenie warunków wejścia do szklarni

#### **Warunki wejścia do szklarni:**

Co jakiś czas każdy proces sprawdza ATOMOWO czy jego REQ, spełnia już warunki wejścia do szklarni:

1. Czy jego własne REQ w kolejce po G i L znajdują się wśród G oraz L żądań o najwyższym priorytecie (dla każdego zasobu G i L adekwatnie), jeśli nie \-\> rezygnuje z tej próby, nie ruszając kolejek
2. Czy jego własne REQ w kolejce po nawóz znajduje się wśród x żądań o najwyższym priorytecie, gdzie x \<= aktualnego licznika nawozu, jeśli nie \-\> rezygnuje z tej próby, nie ruszając kolejek
3. Jeśli do tego czasu proces nie zrezygnował, to wysyła komunikat RELEASE(nawóz), zmniejsza swój lokalny licznik nawozu o 1 i wchodzi do szklarni, gdzie spędza pewien czas t pieląc grządki (wziął porcję nawozu i wszedł do szklarni)

#### **Obsługa obcych REQ:**

**A: gdy proces odbierający czeka w kolejce**

1. Odebranie komunikatu REQ
2. Ulokowanie odebranego REQ w swojej lokalnej kolejce żądań dla każdego zasobu zgodnie z **sortowaniem kolejek**
3. Odesłanie komunikatu ACK ze swoim zaktualizowanym zegarem wektorowym

    **B: gdy proces odbierający (i-ty) pieli grządki**

4. Odebranie komunikatu REQ od procesu j-tego, wraz z dołączonym (v_obcy)
5. Sprawdzenie, czy v_obcy\[j\] ma wyższy priorytet, niż własne żądanie REQ, na podstawie którego aktualnie proces i-ty pieli.

1) jeśli priorytet żądania procesu j-tego jest niższy postępujemy standardowo: Odesłanie komunikatu ACK ze swoim zaktualizowanym zegarem wektorowym
2) jeśli priorytet żądania procesu j-tego jest wyższy, odkładamy otrzymane żądanie na **_stos do obsłużenia po wyjściu ze szklarni_**

#### **Wychodzenie ze szklarni:**

1. Wysłanie komunikatu RELEASE(grabki) oraz RELEASE(łopatki)
2. Obsłużenie REQ oczekujących na **_stosie do obsłużenia po wyjściu ze szklarni_**
3. Udanie się na spoczynek przez losowy czas t

#### **Obsługa obcych RELEASE:**

1. Proces i-ty odbiera od procesu j-tego komunikat RELEASE(typ_zasobu) wraz z zegarem v_obcy.
2. Usunięcie z lokalnej kolejki dot. typ_zasobu wszystkich komunikatów REQ należących do procesu j-tego, których priorytet jest mniejszy niż v_obcy\[j\].
3. Jeśli typ*zasobu to nawóz, proces zmiejsza swoją wartość lokalnego licznika nawozu o 1, jeśli jest to proces kupiec, sprawdza on \*\*\_warunek kupca*\*\*.

#### **Ogrodnik-Kupiec**

Jeden z ogrodników ma przydzieloną funkcję kupca. Ten ogrodnik, gdy zmniejsza swój lokalny licznik nawozu wskutek obsługi komunikatu RELEASE, zawsze sprawdza **_warunek kupca:_**  
 jeśli liczba nawozu po zmniejszeniu wynosi 0, wywołuje on funkcje dostawa \- udaje się do sklepu, kupuje D nawozu (zajmuje to pewną ilość czasu t), następnie wraca i wysyła do wszystkich (w tym do siebie) komunikat DELIVERY

#### **Obsługa DELIVERY:**

każdy proces, po otrzymaniu tego sygnału, zwiększa swój lokalny licznik nawozu o D.
