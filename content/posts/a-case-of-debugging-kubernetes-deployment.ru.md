---
title: "Пример отладки экземпляра Deployment в Kubernetes"
date: 2018-09-09T16:23:22+03:00
draft: false
---

Не так давно я просматривал список багрепортов в репозитории
Kubernetes. Искал что бы такое починить в целях повышения собственной
заметности в сообществе. И наткнулся
[на отчёт](https://github.com/kubernetes/kubernetes/issues/67515),
который следовало бы озаглавить как &laquo;Таинственное обновление
`ReplicaSet` в `Deployment`, которое его ломает&raquo;.

Сейчас причина сбоя видится тривиальной, если подумать, а не использовать
накопившиеся условные рефлексы. Но в тот момент задача показалась
не столь простой, а от того - интересной. И я решил ею заняться.

Раз отчёт об ошибке появился на багтрекере Кубернетиса, то и проблема
скорее всего в самом Кубернетисе, наивно подумал я. Но как вообще
люди решают подобные задачи? Нет ли уже готовой методологии для этого?
Лично я ничего подобного не встречал, но, похоже,
[не я один](https://github.com/kubernetes/kubernetes/issues/67515#issuecomment-415481314)
задаюсь такими вопросами. А потому я решил скопировать свой
комментарий на эту тему из обсуждения отчёта сюда. Так мне его
потом проще будет найти. И, может быть, кому-то ещё пригодится.

<!--more-->

Мой способ отладки это путь проб и ошибок прежде всего, слегка
направляемый некоторым моим прошлым опытом. Но общий алгоритм
таков:

0. Воспроизведи проблему локально. Как правило этот шаг занимает
   больше всего времени.

1. Посмотри во все доступные логи на предмет странного. К сожалению,
   у всех людей разное представление о странном, сильно зависящее от
   опыта. Чем больше опыт, тем очевиднее странное. Мой крохотный опыт
   не позволил найти ничего странного на этом шаге.

2. Попробуй собрать старую версию Кубернетиса. Чем старее, тем лучше.
   Если проблема не воспроизводится, то дальнейшие шаги сводятся
   к последовательным запусками `git bisect` с целью найти коммит,
   который привносит проблему. Но в данном случае проблема воспроизводилась
   даже в Kubernetes v1.8.0, а значит это не обычная регрессия, и надо
   копать дальше.

3. Изучи содержимое проблемных объектов (`ReplicaSet`) посредством
   комманды `kubectl describe ...`. Как объекты одного и того же типа
   отличаются друг от друга? Если какой-то объект меняется с течением
   времени, то следует присмотреться, что именно в нём меняется.
   На этом шаге обнаружилось, что изменяется объект `Deployment` -
   в нём исчезают "tolerations".

4. Найди код, который исполняется в время изменения `Deployment` и
   сделай логгирование по-многословнее. Тут оказалось, что искомый
   код находится в `kube-contoller-manager`, а установка вот такой
   переменной окружения повышает словоохотливость логов
   ```
   LOG_SPEC=deployment_controller*=5,replica_set*=5,sync*=5,recreate*=5
   ```
   Однако единственным результатом этого шага стало подтверждение,
   что объект `Deployment` действительно кем-то изменяется.

5. Измени обработчик обновления объектов `Deployment`, чтобы в логах
   было видно, что конкретно изменяется помимо "tolerations" (например,
   с помощью в [этой библиотеки](https://github.com/d4l3k/messagediff)).
   Тут могут быть сюрпризы... Но не в этот раз.

6. Поищи код, который непосредственно меняет объекты `Deployment`
   через `apiserver` при помощи вот такой команды.
   ```shell
   $ git grep Deployments | grep "Update("
   ```
   В коде Кубернетиса есть всего несколько место, где `Deployment`
   обновляются, но ни одно из них не выглядит подозрительным.
   Это, наконец-то, приводит нас к давно напрашивающемуся выводу о том,
   что источник изменений объекта находится где-то за пределами кода
   самого Кубернетиса.

7. Take a closer look at the YAML files. They seem to be quite complex.

8. Try to make the setup as simple as possible where the issue is still
   reproducible. Here I found that small one line changes in different YAML
   files lead to the issue being not reproducible - the setup is too fragile.

9. Then look through the YAMLs line by line in the hope to find something
   interesting. The interesting bit here is that the deployment operates
   on behalf of the `kube-state-metrics` service account which requests
   permission to do `update` on `Deployment`s.

10. Drop this permission. This confirms that the Deployment doesn't
    change any more.

11. Look into the logs of the running pod. Bingo! In `addon-resizer`'s logs
    there is an error about inability to update the `Deployment`.

12. Try to find the sources of `addon-resizer:1.0`. This turned out to be
    not easy task, because the version 1.0 is **very** old. Apparently the
    culprit is K8s API has evolved too much since v1.0. Looking at `PodSpec`
    used in `addon-resizer:1.0` confirms it.

So, it turned out the problem is in the outdated container image
`quay.io/coreos/addon-resizer:1.0` of the deployment.

It runs a client which updates the `kube-state-metrics` deployment. But the
client uses outdated API which lacks `Tolerations` in `PodSpec`. As result
the updated deployment looses tolerations in its pod template. The deployment
controller recreates `ReplicaSet`. The pods controlled by the new `ReplicaSet`
lack tolerations as well and fail to run on the tainted node.
