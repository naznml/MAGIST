/* =========================================================
   MAGIST — UNIVERSAL RESULTS MODULE
   Один общий файл для платных тестов
========================================================= */

(function () {

  const MAGIST_SUPABASE_URL =
    "https://wbikkbndoaqjqrbolbuw.supabase.co";

  const MAGIST_SUPABASE_KEY =
    "sb_publishable_Z251FafeuejV3jgVgXZFow_W_PLAcTC";


  /* =====================================================
     TOKEN
  ===================================================== */

  const MAGIST_SESSION_TOKEN =
    new URLSearchParams(window.location.search)
      .get("token") || "";


  /* =====================================================
     CONFIG
     Каждый тест может передать свои данные через:
     window.MAGIST_CONFIG
  ===================================================== */

  const CONFIG =
    window.MAGIST_CONFIG || {};


  /* =====================================================
     ЗАЩИТА ОТ ДВОЙНОГО СОХРАНЕНИЯ
  ===================================================== */

  const SAVE_LOCK = {};


  /* =====================================================
     ОСНОВНАЯ ФУНКЦИЯ СОХРАНЕНИЯ
  ===================================================== */

  async function saveResult(data) {

    if (!MAGIST_SESSION_TOKEN) {
      console.warn(
        "MAGIST: token отсутствует."
      );

      return {
        success: false,
        message: "No token"
      };
    }


    if (!data) {
      return {
        success: false,
        message: "No result data"
      };
    }


    const score =
      Number(data.score || 0);

    const maxScore =
      Number(data.maxScore || data.max_score || 0);


    if (
      maxScore <= 0 ||
      score < 0 ||
      score > maxScore
    ) {

      console.warn(
        "MAGIST: неправильный score.",
        score,
        maxScore
      );

      return {
        success: false,
        message: "Invalid score"
      };

    }


    const courseCode =
      data.courseCode ||
      data.course_code ||
      CONFIG.courseCode ||
      CONFIG.course_code ||
      "";


    const bookCode =
      data.bookCode ||
      data.book_code ||
      CONFIG.bookCode ||
      CONFIG.book_code ||
      null;


    const bookTitle =
      data.bookTitle ||
      data.book_title ||
      CONFIG.bookTitle ||
      CONFIG.book_title ||
      null;


    const sectionCode =
      data.sectionCode ||
      data.section_code ||
      null;


    const sectionTitle =
      data.sectionTitle ||
      data.section_title ||
      null;


    const testCode =
      data.testCode ||
      data.test_code ||
      sectionCode ||
      "TEST";


    const testTitle =
      data.testTitle ||
      data.test_title ||
      sectionTitle ||
      testCode;


    const answers =
      Array.isArray(data.answers)
        ? data.answers
        : [];


    /* -----------------------------------------
       Не сохраняем один результат дважды подряд
    ----------------------------------------- */

    const signature =
      JSON.stringify({
        courseCode,
        bookCode,
        sectionCode,
        testCode,
        score,
        maxScore,
        answers
      });


    const now =
      Date.now();


    const lockKey =
      [
        courseCode,
        bookCode,
        sectionCode,
        testCode
      ].join("|");


    const previous =
      SAVE_LOCK[lockKey];


    if (
      previous &&
      previous.signature === signature &&
      now - previous.time < 2000
    ) {

      return {
        success: true,
        skipped: true
      };

    }


    SAVE_LOCK[lockKey] = {
      signature,
      time: now
    };


    /* =====================================================
       ОТПРАВКА В SUPABASE
    ===================================================== */

    try {

      const response =
        await fetch(

          `${MAGIST_SUPABASE_URL}/rest/v1/rpc/save_test_result_by_session`,

          {
            method: "POST",

            headers: {

              apikey:
                MAGIST_SUPABASE_KEY,

              Authorization:
                `Bearer ${MAGIST_SUPABASE_KEY}`,

              "Content-Type":
                "application/json"

            },

            body:
              JSON.stringify({

                p_token:
                  MAGIST_SESSION_TOKEN,

                p_book_code:
                  bookCode,

                p_book_title:
                  bookTitle,

                p_section_code:
                  sectionCode,

                p_section_title:
                  sectionTitle,

                p_test_code:
                  testCode,

                p_test_title:
                  testTitle,

                p_score:
                  score,

                p_max_score:
                  maxScore,

                p_answers:
                  answers

              })

          }

        );


      const result =
        await response.json();


      if (!response.ok) {

        console.error(
          "MAGIST save error:",
          result
        );

        return {
          success: false,
          result
        };

      }


      console.log(
        "MAGIST result saved:",
        result
      );


      return result;

    }

    catch (error) {

      console.error(
        "MAGIST connection error:",
        error
      );


      return {
        success: false,
        error
      };

    }

  }


  /* =====================================================
     УПРОЩЁННАЯ ФУНКЦИЯ ДЛЯ ТЕСТОВ С sections/questions
  ===================================================== */

  function buildAnswersFromSection(
    section,
    selectedAnswers
  ) {

    if (
      !section ||
      !Array.isArray(section.questions)
    ) {

      return [];

    }


    return section.questions.map(
      (question, index) => {

        const selected =
          selectedAnswers?.[index] || [];


        const correct =
          Array.isArray(question.answer)
            ? question.answer
            : Array.isArray(question.answers)
              ? question.answers
              : [];


        const normalizedSelected =
          Array.isArray(selected)
            ? selected
            : [selected];


        const isCorrect =
          normalizedSelected.length === correct.length &&
          normalizedSelected
            .slice()
            .sort()
            .every(
              (value, i) =>
                value ===
                correct.slice().sort()[i]
            );


        return {

          question_id:
            `${section.id || "section"}-q-${index + 1}`,

          topic_code:
            section.id || null,

          topic_title:
            section.title ||
            section.name ||
            section.id ||
            null,

          selected_answer:
            normalizedSelected
              .map(i => question.options?.[i])
              .filter(v => v !== undefined)
              .join(" | "),

          correct_answer:
            correct
              .map(i => question.options?.[i])
              .filter(v => v !== undefined)
              .join(" | "),

          is_correct:
            isCorrect

        };

      }

    );

  }


  /* =====================================================
     СОХРАНИТЬ РЕЗУЛЬТАТ ЦЕЛОГО SECTION
  ===================================================== */

  async function saveSection(
    section,
    selectedAnswers
  ) {

    if (!section) return;


    const answers =
      buildAnswersFromSection(
        section,
        selectedAnswers
      );


    const score =
      answers.filter(
        answer =>
          answer.is_correct === true
      ).length;


    return saveResult({

      courseCode:
        CONFIG.courseCode ||
        CONFIG.course_code,

      bookCode:
        CONFIG.bookCode ||
        CONFIG.book_code,

      bookTitle:
        CONFIG.bookTitle ||
        CONFIG.book_title,

      sectionCode:
        section.id,

      sectionTitle:
        section.title ||
        section.name ||
        section.id,

      testCode:
        section.id,

      testTitle:
        section.title ||
        section.name ||
        section.id,

      score:
        score,

      maxScore:
        answers.length,

      answers:
        answers

    });

  }


  /* =====================================================
     ДЕЛАЕМ MAGIST ДОСТУПНЫМ ДЛЯ КАЖДОГО HTML
  ===================================================== */

  window.MAGIST = {

    token:
      MAGIST_SESSION_TOKEN,

    config:
      CONFIG,

    save:
      saveResult,

    saveSection:
      saveSection,

    buildAnswers:
      buildAnswersFromSection

  };


  console.log(
    "MAGIST Results module loaded"
  );

})();