(ns cloud-itonami.identity.pricing
  "Usage plus a price book -> an invoice. Pure.

  The service that people sign in through knows nothing about money: it emits
  dimensioned usage events (`authn.usage`) and stops. This namespace is the
  only thing that multiplies a measurement by a number, and `pricing.edn` is
  the only place a number lives. A price change is therefore a data edit and
  never a deploy of an authentication service.

  ## An invoice line cites its measurement

  Every line carries the month, the dimension and the quantity it came from.
  An amount a customer cannot trace back to a count is an amount they will
  dispute, and rightly — at renewal, when it is most expensive.

  ## A month nobody measured is not a month of zero

  `:unmeasured` is a distinct outcome from `:billable` with quantity 0. They
  look the same on a bill and mean opposite things: one says nobody used the
  service, the other says we do not know. Collapsing them is how a metering
  outage becomes a credit note six months later."
  (:require [kotoba.lang.text :as str]))

(def billable-dimensions
  "The dimensions the meter actually records. A price book that names anything
  else is refused rather than partially honoured: quoting a number nobody
  counted is worse than refusing to quote."
  #{:mau :verification})

;; ── validating the book ─────────────────────────────────────────────────────

(defn problems
  "What is wrong with a price book, as data. Empty means usable."
  [{:pricing/keys [version currency plans] :as book}]
  (cond-> []
    (nil? book) (conj {:pricing.problem/code :absent})
    (not= 1 version) (conj {:pricing.problem/code :unsupported-version :version version})
    (not (and (string? currency) (= 3 (count currency))))
    (conj {:pricing.problem/code :currency-not-iso-4217 :currency currency})
    (empty? plans) (conj {:pricing.problem/code :no-plans})

    :always
    (into (for [p plans
                :let [ids (concat (keys (:plan/included p)) (keys (:plan/overage p)))]
                d ids
                :when (not (contains? billable-dimensions d))]
            {:pricing.problem/code :unknown-dimension :plan (:plan/id p) :dimension d}))

    :always
    (into (for [p plans
                :when (and (seq (:plan/hard-cap p)) (seq (:plan/overage p)))]
            ;; A plan that both caps and charges past the cap cannot be
            ;; described to a customer in one sentence, which means it will be
            ;; described to them in two different ones.
            {:pricing.problem/code :capped-and-metered :plan (:plan/id p)}))))

(defn usable? [book] (empty? (problems book)))

(defn plan
  [book id]
  (first (filter #(= id (:plan/id %)) (:pricing/plans book))))

;; ── can we sell this at all ─────────────────────────────────────────────────

(defn unmet-gates
  "The gates from ADR-2608110200 決定 4 that are not met."
  [book]
  (vec (remove :gate/met (:pricing/gates book))))

(defn sellable?
  "Whether paid public availability may begin.

  Answered from data rather than from someone remembering. The answer today is
  false, and it should stay easy to ask: the failure this prevents is not a
  wrong invoice but a service sold before it can honour what selling implies."
  [book]
  (and (usable? book) (empty? (unmet-gates book))))

;; ── the fold ────────────────────────────────────────────────────────────────

(defn- overage-quantity
  "Units past what the plan includes. Never negative — an under-used month is
  not a credit."
  [included quantity]
  (max 0 (- (or quantity 0) (or included 0))))

(defn- round-cents [x]
  (/ (Math/round (double (* 100 x))) 100.0))

(defn month-line
  "One month of usage under one plan -> a line, or an `:unmeasured` marker.

  `usage` is a row as `GET /v1/usage` returns it: `{:month :active-users
  :verifications}`. `:active-users` of nil — not 0 — is the shape that means
  the distinct-user keys aged out or were never written."
  [plan {:keys [month active-users verifications] :as usage}]
  (let [dims {:mau active-users :verification verifications}]
    (cond
      (nil? usage) {:line/month month :line/status :unmeasured}

      (every? nil? (vals dims))
      {:line/month month :line/status :unmeasured
       :line/note "no measurement for this month — not the same as no usage"}

      :else
      (let [base (or (:plan/monthly-base plan) 0)
            capped (some (fn [[d cap]]
                           (when (> (or (get dims d) 0) cap) d))
                         (:plan/hard-cap plan))
            items (for [[d rate] (:plan/overage plan)
                        :let [q (overage-quantity (get (:plan/included plan) d)
                                                  (get dims d))]
                        :when (pos? q)]
                    {:item/dimension d
                     :item/quantity q
                     :item/unit-rate rate
                     :item/amount (round-cents (* q rate))})]
        (cond-> {:line/month month
                 :line/status :billable
                 :line/plan (:plan/id plan)
                 ;; The measurement rides along on every line. An amount with
                 ;; no count beside it is unauditable by the person paying it.
                 :line/measured (into {} (remove (comp nil? val) dims))
                 :line/base base
                 :line/items (vec items)
                 :line/amount (round-cents (+ base (reduce + 0 (map :item/amount items))))}
          capped (assoc :line/status :over-cap
                        :line/capped-dimension capped
                        ;; Over the cap on a plan with no overage rate: the
                        ;; answer is "choose a plan", not a surprise invoice.
                        :line/amount (round-cents base)))))))

(defn invoice
  "Usage rows + a price book + a plan id -> the invoice.

  Refuses rather than guesses: an unusable book or an unknown plan produces
  `{:invoice/problems [...]}` and no amounts at all. Half an invoice is worse
  than none, because it looks like a whole one."
  [book plan-id usage-rows]
  (let [ps (problems book)
        p (plan book plan-id)]
    (cond
      (seq ps) {:invoice/problems ps}
      (nil? p) {:invoice/problems [{:pricing.problem/code :unknown-plan :plan plan-id}]}
      :else
      (let [lines (mapv #(month-line p %) usage-rows)
            billable (filter #(#{:billable :over-cap} (:line/status %)) lines)]
        {:invoice/plan plan-id
         :invoice/currency (:pricing/currency book)
         :invoice/lines lines
         :invoice/unmeasured (mapv :line/month (filter #(= :unmeasured (:line/status %)) lines))
         :invoice/total (round-cents (reduce + 0 (map :line/amount billable)))
         ;; Stated on the invoice, not only in the ADR: while a gate is unmet
         ;; this is an estimate of what a month would cost, not a demand for
         ;; payment.
         :invoice/sellable? (sellable? book)}))))

;; ── what a customer is told ─────────────────────────────────────────────────

(defn must-read-disclosures
  "The disclosures a buyer has to see before choosing, in the book's own words."
  [book]
  (vec (filter #(= :must-read (:disclosure/severity %)) (:pricing/disclosures book))))

(defn plan-summary
  "One human line per plan, generated from the same data the invoice uses, so a
  published price list cannot disagree with what is charged."
  [book]
  (for [p (:pricing/plans book)]
    (str (:plan/name p)
         (if-let [b (:plan/monthly-base p)]
           (str " — " (:pricing/currency book) " " b "/month")
           " — contract")
         (when-let [inc (:mau (:plan/included p))]
           (str ", " inc " monthly active users included"))
         (when-let [r (:mau (:plan/overage p))]
           (str ", then " (:pricing/currency book) " " r " each"))
         (when (:mau (:plan/hard-cap p)) ", hard cap")
         (when (not= :proposed (:plan/status p)) "")
         (when (= :proposed (:plan/status p)) " [proposed — not yet offered]"))))
