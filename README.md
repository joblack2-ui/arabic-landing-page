<!DOCTYPE html>
<html lang="ar" dir="rtl">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>الأثر</title>

  <style>
    * {
      box-sizing: border-box;
    }

    body {
      margin: 0;
      min-height: 100vh;
      display: flex;
      align-items: center;
      justify-content: center;
      background: #0b0b0f;
      color: #f5f5f5;
      font-family: Arial, sans-serif;
    }

    .container {
      width: 90%;
      max-width: 520px;
      text-align: center;
    }

    h1 {
      font-size: 48px;
      margin-bottom: 15px;
    }

    p {
      font-size: 18px;
      line-height: 1.8;
      color: #aaa;
      margin-bottom: 35px;
    }

    textarea {
      width: 100%;
      min-height: 150px;
      padding: 16px;
      border: 1px solid #333;
      border-radius: 14px;
      background: #15151b;
      color: white;
      font-size: 17px;
      resize: vertical;
      outline: none;
      margin-bottom: 15px;
      font-family: Arial, sans-serif;
    }

    button {
      border: none;
      border-radius: 14px;
      padding: 16px 32px;
      font-size: 18px;
      cursor: pointer;
      background: white;
      color: #111;
      margin: 5px;
      transition: all 0.2s;
    }

    button:hover {
      opacity: 0.9;
    }

    button:active {
      transform: scale(0.97);
    }

    button:disabled {
      opacity: 0.5;
      cursor: not-allowed;
    }

    #form {
      display: none;
    }

    #saved {
      display: none;
      margin-top: 30px;
      padding: 20px;
      border-radius: 14px;
      background: #15151b;
      color: #ddd;
      line-height: 1.8;
      white-space: pre-wrap;
      text-align: right;
      border: 1px solid #2a2a3a;
    }

    .success {
      color: #77ff77;
      margin-top: 15px;
      font-size: 14px;
    }

    .error {
      color: #ff7777;
      margin-top: 15px;
      font-size: 14px;
    }

    .warning {
      color: #ffbb77;
      margin-top: 15px;
      font-size: 14px;
    }

    .loading {
      color: #aaa;
      margin-top: 15px;
      font-size: 14px;
    }

    .view-saved {
      background: #2a2a3a;
      color: #ddd;
      font-size: 14px;
      padding: 12px 20px;
      margin-top: 15px;
    }
  </style>
</head>

<body>

  <div class="container">

    <h1>الأثر</h1>

    <p>
      بعض الأشياء لا ينبغي أن تصل الآن.
      <br>
      اترك شيئًا للمستقبل.
    </p>

    <button id="startButton" onclick="start()">
      اترك أثرًا
    </button>

    <div id="form">

      <textarea
        id="message"
        placeholder="اكتب أثرك هنا..."
      ></textarea>

      <br>

      <button id="saveBtn" onclick="saveTrace()" disabled>
        حفظ الأثر
      </button>

      <button onclick="cancel()">
        إلغاء
      </button>

      <div id="status"></div>

    </div>

    <div id="saved"></div>

    <button
      id="viewSavedBtn"
      class="view-saved"
      onclick="viewAllSaved()"
      style="display:none;"
    >
      عرض آثارك المحفوظة
    </button>

  </div>

  <script>

    const SUPABASE_URL =
      "https://wkhpgcwevnkmrtnytmbk.supabase.co";

    const SUPABASE_KEY =
      "sb_publishable_lr9IOVlibgbCWNqL3eknLQ_Oe0FIwvR";

    const STORAGE_KEY =
      "traces_backup";

    const MAX_RETRIES = 2;


    document.addEventListener(
      "DOMContentLoaded",
      function () {

        const textarea =
          document.getElementById("message");

        textarea.addEventListener(
          "input",
          function () {

            const saveBtn =
              document.getElementById("saveBtn");

            saveBtn.disabled =
              this.value.trim().length === 0;

          }
        );

        updateViewSavedButton();

      }
    );


    async function saveTrace() {

      const message =
        document
          .getElementById("message")
          .value
          .trim();

      const status =
        document.getElementById("status");

      const saveBtn =
        document.getElementById("saveBtn");


      if (!message) {

        status.textContent =
          "اكتب شيئًا أولًا...";

        status.className =
          "error";

        return;

      }


      saveBtn.disabled = true;

      status.textContent =
        "جارٍ حفظ الأثر...";

      status.className =
        "loading";


      let saved = false;


      try {

        saved =
          await saveToSupabase(message);

      } catch (error) {

        console.error(
          "Supabase error:",
          error
        );

      }


      if (saved) {

        status.textContent =
          "✓ تم حفظ الأثر بنجاح!";

        status.className =
          "success";

      } else {

        status.textContent =
          "حدث خطأ أثناء حفظ الأثر.";

        status.className =
          "error";

      }


      try {

        saveLocally(
          message,
          saved
        );

      } catch (error) {

        console.error(
          "Local backup error:",
          error
        );

      }


      if (saved) {

        document
          .getElementById("message")
          .value = "";

        showSaved({
          message: message
        });

        updateViewSavedButton();

      }


      saveBtn.disabled =
        document
          .getElementById("message")
          .value
          .trim()
          .length === 0;

    }


    async function saveToSupabase(
      message,
      retries = 0
    ) {

      try {

        const controller =
          new AbortController();

        const timeout =
          setTimeout(
            () => controller.abort(),
            10000
          );


        const response =
          await fetch(
            `${SUPABASE_URL}/rest/v1/traces`,
            {
              method: "POST",

              headers: {
                "Content-Type":
                  "application/json",

                "apikey":
                  SUPABASE_KEY,

                "Prefer":
                  "return=representation"
              },

              body: JSON.stringify({
                message: message
              }),

              signal:
                controller.signal

            }
          );


        clearTimeout(timeout);


        if (!response.ok) {

          const errorText =
            await response.text();

          console.error(
            "Supabase HTTP error:",
            response.status,
            errorText
          );

          throw new Error(
            `HTTP ${response.status}`
          );

        }


        const data =
          await response.json();

        console.log(
          "Saved to Supabase:",
          data
        );


        return true;


      } catch (error) {

        console.error(
          "Save attempt failed:",
          retries + 1,
          error
        );


        if (
          retries < MAX_RETRIES
        ) {

          await new Promise(
            resolve =>
              setTimeout(
                resolve,
                1000
              )
          );


          return saveToSupabase(
            message,
            retries + 1
          );

        }


        return false;

      }

    }


    function saveLocally(
      message,
      synced
    ) {

      const traces =
        JSON.parse(
          localStorage.getItem(
            STORAGE_KEY
          ) || "[]"
        );


      traces.push({

        message:
          message,

        timestamp:
          new Date()
            .toISOString(),

        synced:
          synced,

        id:
          Date.now()

      });


      localStorage.setItem(
        STORAGE_KEY,
        JSON.stringify(traces)
      );

    }


    function showSaved(
      trace
    ) {

      if (!trace)
        return;


      const box =
        document.getElementById(
          "saved"
        );


      box.textContent =
        "أثرك المحفوظ:\n\n" +
        trace.message;


      box.style.display =
        "block";

    }


    function updateViewSavedButton() {

      try {

        const traces =
          JSON.parse(
            localStorage.getItem(
              STORAGE_KEY
            ) || "[]"
          );


        const btn =
          document.getElementById(
            "viewSavedBtn"
          );


        if (
          traces.length > 0
        ) {

          btn.style.display =
            "inline-block";

          btn.textContent =
            `عرض آثارك المحفوظة (${traces.length})`;

        }

      } catch (error) {

        console.error(error);

      }

    }


    function viewAllSaved() {

      try {

        const traces =
          JSON.parse(
            localStorage.getItem(
              STORAGE_KEY
            ) || "[]"
          );


        if (
          traces.length === 0
        ) {

          alert(
            "لا توجد آثار محفوظة بعد"
          );

          return;

        }


        let content =
          "آثارك المحفوظة:\n\n";


        traces.forEach(
          (trace, index) => {

            const date =
              new Date(
                trace.timestamp
              ).toLocaleString(
                "ar"
              );


            const status =
              trace.synced
                ? "☁️ محفوظ في السحابة"
                : "💾 محلي فقط";


            content +=
              `${index + 1}. [${date}] ${status}\n` +
              `${trace.message}\n` +
              `${"─".repeat(40)}\n`;

          }
        );


        alert(content);


      } catch (error) {

        console.error(error);

        alert(
          "خطأ في عرض الآثار"
        );

      }

    }


    function start() {

      document.getElementById(
        "form"
      ).style.display =
        "block";


      document.getElementById(
        "startButton"
      ).style.display =
        "none";


      document.getElementById(
        "saved"
      ).style.display =
        "none";


      document.getElementById(
        "status"
      ).textContent =
        "";


      document.getElementById(
        "message"
      ).focus();

    }


    function cancel() {

      document.getElementById(
        "form"
      ).style.display =
        "none";


      document.getElementById(
        "startButton"
      ).style.display =
        "inline-block";


      document.getElementById(
        "status"
      ).textContent =
        "";

    }

  </script>

</body>
</html>
