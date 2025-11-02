// ==== INPUT ====
const lastSemesterConst = 'Summer 2025';
const list_prerequisite_courses = [
  "PRO1014","PRO101","WEB2062","WEB503","WEB3014","AND103","CRO102","GAM102","PRO112","GAM106","GAM202","PRO124",
  "PRO104","SOF102","SOF301","SOF302","DAT203","DAT205","DAT206","SOA103","SOA203","SOA205"
];

const list_studentCodes = ["PH37745","PH39535","PH53624"];

// ==== UTILS ====
const timer = ms => new Promise(res => setTimeout(res, ms));
const clean = (s) => (s || '').replace(/\s+/g, ' ').trim();
const esc = (s) => String(s ?? '')
  .replace(/&/g, '&amp;')
  .replace(/</g, '&lt;')
  .replace(/>/g, '&gt;');

const toNumber = (raw) => {
  const t = clean(raw).replace(',', '.');
  const n = parseFloat(t);
  return Number.isFinite(n) ? n : null;
};

const fmtAvg = (sum, count) => (count > 0 ? (sum / count).toFixed(2) : '-');

// ==== CORE ====
(async () => {
  const textLines = [];
  const renderedItems = [];

  for (const code of list_studentCodes) {
    try {
      let prerequisiteCourse = false; // có môn tiên quyết?
      let lastSemester = false;       // có nợ môn kỳ trước?

      // Thống kê
      let cntChuaDat = 0;
      let cntTruotDD = 0;

      let passSum = 0, passCount = 0;                 // Đạt
      let failSum = 0, failCount = 0;                 // Chưa đạt + Trượt điểm danh

      const url = `https://gv.poly.edu.vn/student/view_transcript_detail_student?student_code=${encodeURIComponent(code)}`;
      const res = await fetch(url, { credentials: 'include' });
      const html = await res.text();
      const doc = new DOMParser().parseFromString(html, 'text/html');

      const rows = doc.querySelectorAll('table.table.table-bordered.table-hover tbody tr');

      const badSubjects = [];

      rows.forEach(row => {
        const cells = row.querySelectorAll('td');
        if (cells.length >= 12) {
          const tenMon = clean(cells[3]?.innerText);
          const hocKyRaw = clean(cells[2]?.innerText);
          const maMon = clean(cells[4]?.innerText);
          const trangThaiRaw = clean(cells[10]?.innerText);
          const diemRaw = cells[6]?.innerText;                 // Cột 7 (index 6)
          const diem = toNumber(diemRaw);

          // Cộng dồn điểm theo trạng thái (chỉ khi có số)
          if (trangThaiRaw === 'Đạt') {
            if (diem !== null) { passSum += diem; passCount++; }
          } else if (trangThaiRaw === 'Chưa đạt' || trangThaiRaw === 'Trượt điểm danh') {
            if (diem !== null) { failSum += diem; failCount++; }
          }

          // Ghi nhận các môn nợ để liệt kê (loại trừ Đạt / Đang học / Chưa học)
          if (
            trangThaiRaw !== 'Đạt' &&
            trangThaiRaw !== 'Đang học' &&
            trangThaiRaw !== 'Chưa học'
          ) {
            // Đánh dấu thống kê số lượng theo loại nợ
            if (trangThaiRaw === 'Chưa đạt') cntChuaDat++;
            if (trangThaiRaw === 'Trượt điểm danh') cntTruotDD++;

            // Đánh dấu nếu là môn tiên quyết
            if (list_prerequisite_courses.includes(maMon)) {
              prerequisiteCourse = true;
              badSubjects.push(`[${tenMon}]`);
            }
            // Đánh dấu nếu là nợ từ kỳ trước
            else if (hocKyRaw === lastSemesterConst) {
              lastSemester = true;
              badSubjects.push(`||${tenMon}||`);
            }
            // Mặc định
            else {
              badSubjects.push(tenMon);
            }
          }
        }
      });

      const debtCount = badSubjects.length;
      const header = `${code}(${debtCount})`;
      let lineText = debtCount > 0
        ? `${header} , ${badSubjects.join(' , ')}`
        : `(0)`;

      if (prerequisiteCourse) { lineText += `, Tiên quyết`; }
      if (lastSemester) { lineText += `, 50%`; }

      // Tính TB
      const avgPass = fmtAvg(passSum, passCount);
      const avgFail = fmtAvg(failSum, failCount);

      // Phần bổ sung thống kê (ngăn bằng TAB \t)
      // Format: [TAB] Tổng Chưa đạt [TAB] Tổng Trượt điểm danh [TAB] TB Đạt [TAB] TB (Chưa đạt + Trượt)
      lineText += `\t${cntChuaDat}\t${cntTruotDD}\t${avgPass}\t${avgFail}`;

      textLines.push(lineText);
      renderedItems.push({ header, subjects: badSubjects });

      await timer(500);
    } catch (err) {
      console.error(`Lỗi xử lý ${code}:`, err);
      const header = `${code}(0)`;
      // Khi lỗi, phần thống kê sẽ để "-": chưa có dữ liệu
      textLines.push(`${header} , [LỖI KHI TẢI DỮ LIỆU]\t-\t-\t-\t-`);
      renderedItems.push({ header, subjects: ['[LỖI KHI TẢI DỮ LIỆU]'] });
    }
  }

  const output = textLines.join('\n');

  // ===== HIỂN THỊ KẾT QUẢ =====
  document.open();
  document.write(`
    <div style="font-family: Arial, sans-serif; padding:16px; max-width:960px; margin:0 auto;">
      <h2 style="margin:0 0 12px 0;">Kết quả tổng hợp</h2>
      <p style="margin:0 0 8px 0;">(Khối dưới để copy sang Excel/Google Sheets — mỗi dòng là 1 ô/cột do dùng tab)</p>
      <textarea id="outTA" style="width:100%; height:30vh; font-family: Consolas, monospace; font-size:13px; padding:8px;">${output}</textarea>
      <div style="margin-top:8px;">
        <button id="btnCopy" style="padding:6px 10px; cursor:pointer;">Copy toàn bộ</button>
      </div>
    </div>
    <script>
      document.getElementById('btnCopy')?.addEventListener('click', async () => {
        const ta = document.getElementById('outTA');
        ta.select();
        ta.setSelectionRange(0, ta.value.length);
        try {
          await navigator.clipboard.writeText(ta.value);
          alert('Đã copy!');
        } catch (e) {
          document.execCommand('copy');
          alert('Đã copy (fallback)!');
        }
      });
    </script>
  `);
  document.close();
})();
