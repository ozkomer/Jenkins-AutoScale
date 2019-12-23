# Atolye15 Demo

Bu ekrana sadece Hello world yazdiran basit bir Nest.JS uygulamasi. Senden asagidaki sekilde bir pipeline olusturmani bekliyoruz;

Git repomuzda `master` ve `develop` branch'leri bulunuyor. Insanlar `develop` branch'ine feature branch'ler uzerinden yeni ozellikler ekleyebilirler. Bunun icin de soyle bir pipeline planliyoruz;

Kisi gerekli commit'leri attiktan sonra CI (Biz genelde CircleCI kullaniyoruz ama sen istedigini kullanabilirsin) aracinda sirasiyla asagidaki kontroller calisir;

- Lint kurallari geciyor mu: `yarn lint`
- Formatlama kurallarina uyuyor mu: `yarn format:check`
- Unit testler geciyor mu: `yarn test`
- Coverage threshold'una uyulmus mu?: `yarn test:cov`
- E2E testler geciyor mu: `yarn test:e2e`

Tum bu kontroller pass olduktan sonra PR `develop` ile birlestigi zaman senin yazmis oldugun bu projeye dahil edecegin `Dockerfile`'daki stepleri takip eden herhangi bir builder'da image build alip onu herhangi bir private container registy'e yollamani bekliyoruz. GCloud'ta oldugunu varsayarsak bu araclar cloud build ve GCR olacaktir. Sen istedigin cozumu kullanabilirsin.

Image registry'e gittikten sonra latest tag'li bu image'in Kubernetes tarafinda senin yazdigin manifestolara uygun olarak `stage` namespace'inde yayina girmesini istiyoruz. Bu asamada ilgili kisiye mail gidebilir. Daha sonra Git tarafinda `develop`'tan `master`'a PR acildiginda tum surec tekrar yukaridaki gibi isleyip en sonunda `production` namespace'inde Kubernetes uzerinde yayinda olmasini bekliyoruz.

Kubernetes tarafinda da Let's encrypt uzerinden auto provision ile SSL ayarlarsan da super olur.

Pipeline'in istedigin kismini es gecebilir veya kendince daha dogru oldugunu dusundugun bir hale getirebilirsin.

NOT: Dependency'lerin kurulmasi icin proje dizininde `yarn` komutunun calistirilmasi gerekiyor.

NOT: Uygulama `yarn start:prod` komutu ile ayaga 3000 portunda ayaga kalkiyor.
