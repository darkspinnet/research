# Hypno Mantis

The 2026-09-06 close-range reports show CryosSpecialThree object 78 repeatedly selecting RezMelee at about 3.18 units while first-action recovery reports SleepMushroom's 14.2-unit range. Secondary pursuit rejected the melee family because the noun's default is projectile, released its action, and restarted that loop. RezMelee is now an admitted secondary pursuit, retaining its 1.5-unit melee profile until arrival.

SleepMushroom now records its existing authored profile cooldown separately from its 900 ms animation release. After release, the ordinary selector can pursue and use RezMelee while the pod cooldown runs. Melee completion returns to that selector rather than permanently choosing melee. The released projectile and cloud retain their own lifetime independently of subsequent attacks. Session reset clears the pod readiness map.
