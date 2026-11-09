# Inventaire des événements du domaine qui pourraient déclencher une Push Notification

Ticket : dr1rrb/training-aihero-cohort005#2 (enfant de la map #1).
Date : 2026-11-09.

## Question

Quels événements existent aujourd'hui dans Cadence qui concernent **une autre personne que celle qui agit** ? Pour chacun : qui le déclenche, qui est concerné, où il est écrit dans le code, et s'il envoie déjà un email.

Ce document est un inventaire. Il ne recommande pas d'événements : le catalogue sera choisi plus tard.

## Sources et provenance

Toutes les affirmations viennent de la lecture directe du code ; il n'existe pas de doc tierce sur ce sujet. Les rôles suivent `GLOSSARY.md` (Student, Instructor, Platform Admin, Team Admin).

**Attention à la provenance.** Le code lu est celui de la copie de travail principale au 2026-11-09. Une partie n'est pas encore commitée sur `origin/main` (commit `d241a58`) : `app/lib/refunds.server.ts`, `app/lib/mailer.server.ts`, `app/routes/admin.refunds.tsx`, `GLOSSARY.md`, `docs/adr/`, ainsi que des changements dans `courses.$slug.refund.ts`, `redeem.$code.tsx`, `schema.ts` et d'autres fichiers. Les numéros de ligne renvoient à cette copie de travail. Ils pourront bouger une fois ce travail commité.

Méthode : relevé de toutes les écritures en base (`.insert(` / `.update(` / `.delete(` dans `app/`, hors tests), rattachement de chaque écriture à l'action de route qui l'appelle, puis relevé de tous les appels à `sendEmail` (`grep sendEmail app`).

## Correspondance entre rôles et code

- `UserRole` (`app/db/schema.ts:11`) vaut `student | instructor | admin`. `admin` correspond au **Platform Admin**.
- Le **Team Admin** n'est pas un `UserRole`. C'est une ligne `team_members` avec `role = admin` (`TeamMemberRole`, `schema.ts:40`), créée par `getOrCreateTeamForUser` lors d'un achat de Team Licence (`app/routes/courses.$slug.purchase.tsx:210`).
- « Staff » d'un cours (`app/lib/access.server.ts:99`) : Platform Admin, ou Instructor propriétaire du cours (`courses.instructorId`).

## Emails envoyés aujourd'hui

`app/lib/mailer.server.ts` expose une seule fonction, `sendEmail` : Postmark si `POSTMARK_SERVER_TOKEN` est défini, sinon un fichier dans `.outbox/`. Il y a **exactement deux appelants** :

1. `app/routes/courses.$slug.refund.ts:75` : après `raiseRefundRequest`, un email d'accusé de réception **au buyer lui-même** (« Refund requested — … », `app/lib/refunds.server.ts:394-406`).
2. `app/routes/admin.refunds.tsx:130` : après `approveRefundRequest` ou `declineRefundRequest`, un email **au buyer** (« Refund approved/declined — … », `buildDecisionEmail`, `refunds.server.ts:736`).

Aucun autre événement n'envoie d'email. `enrollUser` (`courses.$slug.purchase.tsx:145`) a un paramètre `sendEmail`, mais la branche est vide (`// Would send welcome email here`), et l'appelant passe `false` (`:373`).

## Inventaire : événements qui touchent une autre personne

| # | Événement | Déclenché par | Personne(s) concernée(s) (≠ acteur) | Où (fichier : fonction) | Email aujourd'hui |
|---|---|---|---|---|---|
| 1 | Question posée sur une leçon (comment de premier niveau) | Student enrollé (ou staff) | Instructor propriétaire du cours (et Platform Admins), qui la voient dans leur file « Unanswered questions » | `app/routes/courses.$slug.lessons.$lessonId.tsx` : `action` intent `create-comment` → `createComment` (l. 404), avec `parentId = null` | Non |
| 2 | Réponse à un comment (reply), depuis la page leçon | Toute personne ayant accès au contenu | Auteur du comment parent (souvent un Student) ; Instructor du cours si l'auteur de la réponse est un Student | même fichier : `createComment`, avec `parentId` renseigné | Non |
| 3 | Réponse de l'Instructor ou d'un Platform Admin depuis la file de questions | Instructor propriétaire ou Platform Admin (`requireStaff`) | Student auteur de la question | `app/routes/instructor.questions.tsx` : `action` → `createComment` (l. 118) | Non |
| 4 | Comment supprimé par le staff (soft delete) | Instructor propriétaire ou Platform Admin (`isStaff`) | Auteur du comment | `courses.$slug.lessons.$lessonId.tsx` : intent `delete-comment` → `deleteComment` (l. 459) | Non |
| 5 | Refund Request ouverte (Open Refund Request) | Buyer : Student (Individual Purchase) ou Team Admin (Team Licence) | Platform Admins, seuls à décider (GLOSSARY). Pour une Team Licence, les Unclaimed Seats sont gelés (ADR 0003) : un Student qui essaie d'en réclamer un est refusé (`SEAT_ON_HOLD`) | `app/routes/courses.$slug.refund.ts` : `action` → `raiseRefundRequest` (`app/lib/refunds.server.ts:357`) | Oui, mais **seulement au buyer** (accusé de réception). Aucun Platform Admin n'est prévenu |
| 6 | Refund Request approuvée (Approved Refund Request) | Platform Admin | Buyer (Student ou Team Admin). Individual Purchase : le buyer perd son Enrollment, sauf s'il a aussi un Claimed Seat (ADR 0002, 0004). Team Licence : les Unclaimed Seats deviennent définitivement inutilisables | `app/routes/admin.refunds.tsx` : `action` → `approveRefundRequest` (`refunds.server.ts:614`) | Oui, au buyer (`buildDecisionEmail`) |
| 7 | Refund Request déclinée (Declined Refund Request) | Platform Admin | Buyer ; pour une Team Licence, les Seats redeviennent réclamables | `admin.refunds.tsx` : `action` → `declineRefundRequest` (`refunds.server.ts:692`) | Oui, au buyer, avec la note obligatoire |
| 8 | Nouvelle Enrollment via Individual Purchase | Student (pour lui-même) | Instructor du cours (nouvel inscrit) | `app/routes/courses.$slug.purchase.tsx` : `action` intent `confirm-purchase` → `createPurchase` (l. 180) + `enrollUser` (l. 145) | Non (le stub d'email de bienvenue est vide) |
| 9 | Achat d'une Team Licence | Team Admin (acheteur) | Instructor du cours (vente). Aucun Student n'est encore concerné : les Seats ne sont pas attribués à des personnes | `courses.$slug.purchase.tsx` : intent `confirm-team-purchase` → `createTeamPurchase` (l. 245), qui génère les coupons | Non |
| 10 | Seat réclamé (Claimed Seat) → nouvelle Enrollment | Student | Team Admin qui a acheté la Team Licence ; Instructor du cours (nouvel inscrit). Une fois un Seat réclamé, la Team Licence ne peut plus être remboursée (ADR 0003) | `app/routes/redeem.$code.tsx` : `action` → `redeemCoupon` (l. 47) | Non |
| 11 | Note sur un cours (création ou modification) | Student enrollé | Instructor du cours | `app/routes/courses.$slug.tsx` : `action` → `rateCourse` (l. 330) | Non |
| 12 | Tentative de quiz soumise (réussie ou non) | Student | Instructor du cours (qui voit les meilleurs scores dans son éditeur, `instructor.$courseId.tsx` `getBestAttempt` l. 466) | `courses.$slug.lessons.$lessonId.tsx` : intent `submit-quiz` → `computeResult` (l. 531), insert `quizAttempts` | Non |
| 13 | Leçon marquée terminée | Student | Instructor (progression des Students, `instructor.$courseId.students.tsx`) | même fichier : intent `mark-complete` → `markLessonComplete` (l. 218) | Non |
| 14 | Changement de statut d'un cours (Draft → Published, → Archived, …) | Instructor propriétaire ou Platform Admin | Published : Students en général (le cours apparaît dans le catalogue, `home.tsx:115`, `courses.tsx:288`). Archived : Students enrollés (le cours n'est plus achetable, `courses.$slug.purchase.tsx:286`). Changement par un Platform Admin : l'Instructor propriétaire | `app/routes/instructor.$courseId.tsx` : intent `update-status` → `updateCourseStatus` (l. 154) ; `app/routes/admin.courses.tsx` : intent `update-status` → `updateCourseStatus` (l. 76) | Non |
| 15 | Contenu ajouté ou modifié dans un cours (module, leçon, contenu de leçon) | Instructor propriétaire ou Platform Admin | Students enrollés | `instructor.$courseId.tsx` : `createModule`, `createLesson` (l. 265), `deleteLesson` (l. 304), `deleteModule` (l. 236), etc. ; `instructor.$courseId.lessons.$lessonId.tsx` : `updateLesson` (l. 66) | Non |
| 16 | Quiz supprimé ou modifié | Instructor propriétaire ou Platform Admin | Students : `deleteQuiz` supprime aussi leurs `quizAttempts` et `quizAnswers` | `app/routes/instructor.$courseId.lessons.$lessonId.quiz.tsx` : `deleteQuiz` (l. 201), `updateQuiz` (l. 180) | Non |
| 17 | Changement de rôle ou d'identité d'un utilisateur | Platform Admin | L'utilisateur modifié (par exemple un Student promu Instructor) | `app/routes/admin.users.tsx` : intents `update-role` → `updateUserRole` (l. 60), `update-user` → `updateUser` | Non |
| 18 | Prix ou PPP d'un cours modifié | Instructor propriétaire ou Platform Admin | Aucun destinataire précis (futurs acheteurs) | `instructor.$courseId.tsx` : `updateCoursePrice`, `updateCoursePppEnabled` | Non |

## Événements de l'exemple du ticket qui n'existent pas dans le code

- **Refund émis / Refund échoué (Issued Refund, Failed Refund, Pending Refund).** Ces termes sont dans `GLOSSARY.md`, mais rien ne les modélise. La table `refunds` (`schema.ts:232`) n'a que `status ∈ {open, approved, declined}` et `amountRefunded`. Il n'y a ni état de paiement ni intégration de paiement : l'approbation fixe `amountRefunded = amountPaid`, et l'email dit seulement que l'argent « is on its way back ». Un tel événement push supposerait d'abord un modèle et une source (webhook du prestataire de paiement) qui n'existent pas.
- **Fin d'un cours (Enrollment terminée).** `enrollments.completedAt` existe et il est lu (`dashboard.tsx:213`, `instructor.$courseId.students.tsx`), mais **aucun code ne l'écrit**. Aucune écriture `update(enrollments)` hors tests.

## Événements sans tiers concerné (exclus, pour mémoire)

Écritures qui ne touchent que l'acteur lui-même, ou de la configuration :

- `signup` (création d'un Student) ;
- `settings` (nom et bio) ;
- `api.video-tracking` (événements de lecture, à la base du Watched) ;
- édition ou suppression de son propre comment (`updateComment`) ;
- `admin.categories` (CRUD de catégories) ;
- `instructor.new` (création d'un cours en Draft, invisible des autres).

## Conditions temporelles (pas des événements)

Plusieurs règles reposent sur l'écoulement du temps. Aucune écriture ne les déclenche, et il n'y a ni planificateur ni file de jobs (la map #1 le note déjà) :

- fin de la fenêtre de remboursement : `lastDayOfWindow` (`refunds.server.ts:52`) ;
- dépassement de la Watch Limit : calculé à la volée depuis `video_watch_events` ;
- question restée sans réponse : `STALE_AFTER_DAYS = 3` (`instructor.questions.tsx`), aujourd'hui un simple marqueur visuel.

Une push sur l'une de ces conditions demanderait un mécanisme d'exécution différée qui n'existe pas.

## Constats factuels utiles pour la suite

- **Recouvrement avec l'email.** Seuls les événements 5, 6 et 7 envoient un email, et toujours au buyer. Tous les autres ne préviennent personne aujourd'hui.
- **Aucun Platform Admin n'est prévenu d'une Open Refund Request.** Il doit ouvrir `/admin/refunds` pour la voir. C'est le seul événement où le décideur désigné par le GLOSSARY ne reçoit aucun signal.
- **Les questions non répondues** n'ont qu'une file à consulter (`/instructor/questions`). Les réponses du staff n'arrivent au Student que s'il revient sur la leçon.
- **Déclenchement synchrone.** Tous les événements se produisent dans l'`action` d'une route (requête HTTP du déclencheur). Les fonctions de refunds renvoient déjà l'email à envoyer comme valeur (`result.email`), et la route l'envoie après la transaction. Une push pourrait suivre le même chemin.
- **Désignation des destinataires.** Pour un comment, retrouver l'Instructor demande `lesson → module → course.instructorId` (déjà fait par `getCourseForLesson`). « Platform Admins » veut dire tous les `users.role = admin`. Le Team Admin d'une Team Licence est `purchases.userId` de la Purchase du coupon.
