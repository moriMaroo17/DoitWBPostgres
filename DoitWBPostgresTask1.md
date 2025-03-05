# Эпиграф
> Все делаю через докер + GUI
# Подготовка
1. Стянул постгрю с докерхаба
`$ docker run -p 5432:5432 --name some-postgres -e POSTGRES_PASSWORD=mysecretpassword -d postgres
`
2. Обновил репы apt, поставил curl
`$ apt update`
`$ apt install curl`

# Действия по заданию
1. Стащил архив с гитахаба
`$ cd /home && mkdir downloads && cd downloads`
`$ curl https://storage.googleapis.com/thaibus/thai_small.tar.gz --output thai_small.tar.gz`
2. Загрузил архив в БД
`$ tar -xf thai_small.tar.gz`
`$ su postgres`
`$ psql < thai.sql`
3. Подключился к контейнеру с gui (Antares SQL)
4. Посчитал кол-во поездок
`select count(*) from book.tickets;` -> получил `5185505`
